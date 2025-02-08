---
layout: post
title:  "Debugging esp-hal, part 1 - The mysteriously hanging display drivers"
date:   2025-02-15 21:32 +0100
categories: rust esp-hal
excerpt_separator: <!--more-->
---

## Preface

This post is the first of (maybe hopefully, maybe not?) many writeups detailing esp-hal bugs and
their fixes. Some of these are probably very obvious in hindsight, but maybe the story itself may
be interesting. Or embarassing. Or full of learning opportunities. Regardless, journey before
destination.

A short while ago, one of our prominent community members sent me a project, with a short message
that two of his display drivers don't work on two separate ESP32 microcontrollers (ESP32, ESP32-S3),
using two separate peripherals (I2S on the ESP32, LCD_CAM on the ESP32-S3), with esp-hal driver that
implement the I8080 display format. That's pretty weird because their respective drivers don't have
much code in common, but maybe this will be an easy case, and I can get two for one.

*Narrator: It was not.*

<!--more-->

## The reproducer

The reporter has kindly given me a reproducer project and detailed description on how to run that
project - thank you, very helpful! It's one step below "just needs `cargo run`" (the less I have to
modify, the fewer chances I have to accidentally make the reproducer working - and thus, not a
reproducer), but as the project targets multiple chips, it's understandable.

The project is a driver for HUB75 displays, targeting ESP32 chips. Besides the two MCUs with the
issues (ESP32 and ESP32-S3), it also supports ESP32C6 via its PARL_IO peripheral, which - and I had
no reports of the contrary - is hopefully working as it should.

The project contains an example for each of the drivers, and the examples do more-or-less the
same thing. The examples set up two embassy executors, and each executor runs a single task. The
executors run at different priority levels. The lower priority executor runs the `display` task
which generates frame buffers, while the other executor runs the `hub75` task which transfers
the frame buffers to the display. The example implements double buffering by swapping two frame
buffers between the tasks.

The display drivers use DMA (Direct Memory Access) to transfer data from memory to the peripheral.
The DMA frees up the CPU to execute other code while it's handling the data transfer in the
background. As an aside, this gives us a straight forward way to implement asynchronous peripheral
drivers: we can fire off a transfer, then periodically* check if the transfer has finished, and in
the mean time, the CPU can take care of other tasks.

(*: or even better, we can register an interrupt handler to notify us!)

The project contains a log statement in one of the periodically called functions, which gave me an
easy way to tell if the driver was working, or locking up.

## I2S

All right, it was time to see if I can reproduce the issue. I configured the project for ESP32,
downloaded it do my hardware, and it failed exactly as I had expected it to. The "periodic" log
message was printed once, then nothing.

As we have some hardware-in-the-loop tests and examples which haven't indicated anything wrong
before, I wasn't sure if this is a regression in esp-hal, or the use case has never really worked.
Because of this, I did not try to bisect this issue (turns out, I may have found the issue sooner
with a bisect. Thank you sharp hindsight...). The project was set up to use `espflash` instead of
`probe-rs`, and instead of spending time on setting up the debugger (I know, I know), I have instead
started inserting log lines to narrow down where exactly the program stops.

I compiled, flashed, and.... the program did not halt. Weird. Backtracking, I found that placing a
log message between the `hub75_task`'s `render` and `wait_for_done` functions, the example just
magically started working. We've had a few places in esp-hal, where we needed to insert microsecond
waits for the DMA (which is actually an inadequate workaround) before starting a peripheral
transfer, but this is _after_ we start the transfer, so what gives?

So I needed to dig deeper into esp-hal, and the I2S driver.

Our DMA-driven `async` drivers work by setting up the DMA transfer, then starting the peripheral,
then creating a `Future` object, and polling it. The future's implementation looks something like
this (the example is a bit simplified):

```rust
impl<TX> core::future::Future for DmaTxFuture<'_, TX>
where
    TX: Tx,
{
    type Output = Result<(), DmaError>;

    fn poll(
        self: core::pin::Pin<&mut Self>,
        cx: &mut core::task::Context<'_>,
    ) -> Poll<Self::Output> {
        self.tx.waker().register(cx.waker());
        if self.tx.is_done() {
            Poll::Ready(Ok(()))
        } else if self.has_errors() {
            Poll::Ready(Err(DmaError::DescriptorError))
        } else {
            // The interrupt handler will wake the task using the waker we registered above.
            self.tx.listen_for_interrupts();
            Poll::Pending
        }
    }
}
```

Essentially, this future does nothing if the transfer is already done the first time it is polled.
This is why our log statement made the driver "work": we injected enough delay between `render` and
`wait_for_done` for the transfer to complete, bypassing the whole `async` machinery.

But what does this mean for us? We know that the transfer is set up and started correctly - it can
actually finish, because `is_done()` can return `true` if we wait long enough. What remains is one
of the following:

- The DMA does not actually trigger the interrupt request.
- The interrupt handler has a bug.
- We do not handle the interrupt request. This means we either
  - do not register the interrupt handler,
  - or we do not start listening for the interrupt request.

At first glance, any of these issues should break other peripherals, too, since we are using the
same DMA driver in other places of esp-hal as well. So let's trace through esp-hal to see what's
happening!

First, we need to set up our peripheral for `async` operation. This involves setting up the
relevant interrupt handlers, which is one of the important bits we need to double-check here.

```rust
pub fn into_async(self) -> I2sParallel<'d, Async> {
    I2sParallel {
        instance: self.instance,
        tx_channel: self.tx_channel.into_async(),
        _guard: self._guard,
    }
}
```

Okay, this isn't much help - we're just transforming the DMA driver into an `async` DMA driver.
We do that in a lot of places, what could go wrong?

*Narrator: this is called foreshadowing.*

So how do we set up the DMA for `async` operation? Let's drill down into the function!

```rust
/// Converts a blocking channel to an async channel.
pub(crate) fn into_async(mut self) -> ChannelTx<'a, Async, CH> {
    if let Some(handler) = self.tx_impl.async_handler() {
        self.set_interrupt_handler(handler);
    }
    self.tx_impl.set_async(true);
    ChannelTx {
        tx_impl: self.tx_impl,
        _phantom: PhantomData,
        _guard: self._guard,
    }
}
```

Here we set the interrupt handler, and set a flag (`set_async(true)`). The flag is meant to
ensure that we only handle interrupts that belong to `async` code. On later chips, the DMA TX and RX
channels are independent, so we need to tell their interrupt handlers (it's complicated, some chips
have a single handler for DMA channels, others have two) which ones need to be handled by the
driver, and which ones do not.

Okay, but why is `set_interrupt_handler` inside a condition? Well, it comes back to the DMA hardware
and the assumptions we made based on that hardware. On later chips, we have separate DMA RX and TX
interrupt handlers, so we need to register them separately. Those TX and RX halves can also be used
freely, so we can't just register the TX channels' interrupt handler when we set up the RX channel.
On the ESP32, this is a bit different. We can only use the I2S DMAs for the I2S peripherals, and
their RX/TX channels aren't independent - or, that is what we thought.

If we look into what we do for the [TX channel][1], we immediately see what's wrong:

```rust
fn peripheral_interrupt(&self) -> Option<Interrupt> {
    None
}

fn async_handler(&self) -> Option<InterruptHandler> {
    None
}
```

We have assumed that the TX and RX channels would be used together. Setting up the RX channel would
have configured the interrupt handler (because there is only one). We thought this adequate, and
that we can save the CPU time spent on the TX-side setup. Our I2S tests were passing*, so I did not
suspect the DMA driver to be the culprit for a while. However, we were wrong in our assumption - the
I2S parallel display driver (which was introduced some time later) only uses the TX half of the DMA,
which simply did not configure its interrupt handler correctly.

(*: more specifically, the tests are differently broken on ESP32 - but don't mind that for now.)

Returning the expected interrupt handler and peripheral interrupt from these functions solved this
issue. I've also added a test to our suite to ensure we don't break this code again, at least in
the same way.

# LCD_CAM (I8080 driver)

Unfortunately, the above fix (obviously) did not resolve the LCD_CAM issue on the ESP32-S3. My
"log trick" also did not help at all, and we even have an I8080 display driver [example][2]
in esp-hal that just works (although that example uses blocking code, not `async`)! This basically
left me back at square 1, except the peripheral worked in the examples, but not in user code - so
maybe square 0?

On a surface level, the problem is similar: we can trace through the code, and the same things
happen (or, rather, don't happen):

- The last log line in the user code that prints anything is between `render` and `wait_for_done`.
- The interrupt handler does not fire.

The peripheral driver is structurally similar to the I2S display driver. Sending data involves
some peripheral setup, followed by DMA configuration, starting the DMA transfer, then starting
the peripheral transfer.

But on the ESP32-S3, our DMA driver doesn't have the same bug! The DMA channels are freely mixable
between peripherals (meaning every peripheral uses the same DMA driver), and we set up and handle
their interrupts correctly (otherwise other peripheral driver would be broken too, probably). The
DMA channels are also not tied to peripherals as they are on the ESP32 - there is a single channel
implementation. Looking through the code, both [TX][3] and [RX][4] channels should correctly set up
their [interrupt handlers][5]. Yet the interrupt request doesn't fire in user code.

Another difference is that, in this case the driver uses its own interrupt, and not the DMA's
interrupt to check for completion. The `poll` implementation and related bits look like this:

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    LCD_DONE_WAKER.register(cx.waker());
    if Instance::is_lcd_done_set() {
        Poll::Ready(())
    } else {
        Instance::listen_lcd_done();
        Poll::Pending
    }
}

impl Instance {
    fn enable_listenlcd_done(en: bool) {
        LCD_CAM::regs()
            .lc_dma_int_ena()
            .modify(|_, w| w.lcd_trans_done_int_ena().bit(en));
    }

    pub(crate) fn listen_lcd_done() {
        Self::enable_listenlcd_done(true);
    }

    pub(crate) fn unlisten_lcd_done() {
        Self::enable_listenlcd_done(false);
    }

    pub(crate) fn is_lcd_done_set() -> bool {
        LCD_CAM::regs()
            .lc_dma_int_raw()
            .read()
            .lcd_trans_done_int_raw()
            .bit()
    }
}
```

This again, looks reasonable. We check for completion, start listening for an interrupt if we have
to, and we even register the waker as we are supposed to. Yet the interrupt request doesn't fire in
user code.

So, since the esp-hal example works, maybe we can find some difference between the two
implementations if we look at it close enough. Let's start with how the blocking version
waits for completion:

```rust
pub fn is_done(&self) -> bool {
    self.i8080
        .lcd_cam
        .lcd_user()
        .read()
        .lcd_start()
        .bit_is_clear()
}

pub fn wait(mut self) -> (Result<(), DmaError>, I8080<'d, Dm>, BUF) {
    while !self.is_done() {}

    // [... Clear "done" interrupt.]

    // [... reconstruct the driver and return]
}
```

This is different from the `async` version, which looks at the DMA interrupt status bit. So I tried
changing the `async` implementation to also sample the `lcd_start` bit (because I knew it was a
valid way of detecting completion), then changed `poll` to immediately wake the caller task (so
it will loop regardless of interrupts occurring or not), and still, nothing happened.

Next, I tried printing what happens to the `lcd_start` bit. As we are transferring a large-ish
amount of data, I expected `lcd_start` to stay `true` for a few iterations, then turn `false`.
However, the bit never went `true` in the first place. Which is especially weird considering we are
the ones writing `true` into it in the first place!

This suggested that the transfer wasn't configured properly, but configuration is done by the same
code that the blocking version calls, and we know that works. I went on to printing a few random
register values and that is when I noticed something very suspicious: all registers were 0, even
after writing to them!

Usually, to write a register, we need two conditions to hold:
- The peripheral must not be held in reset.
- The peripheral must have a running clock signal.

In esp-hal, we manage peripheral resets and clock signals by a mechanism we call `PeripheralGuard`.
When created, a guard object enables clocks and takes the peripheral out of reset, and when dropped,
disables the peripheral it manages. This mechanism allows us to save power by only keeping awake
what the user actually uses*.

(*: We could probably do better, by dynamically disabling clocks between transfers. Disabling
peripheral clocks shouldn't have adverse effects if the driver isn't actively in use, but this
needs to be verified)

```rust
// Somewhat simplified excerpt
pub(crate) struct PeripheralGuard {
    peripheral: Peripheral,
}

impl PeripheralGuard {
    pub(crate) fn new(p: Peripheral) -> Self {
        if PeripheralClockControl::enable(p) {
            // Reset the peripheral when we create the first guard object for it.
            PeripheralClockControl::reset(p);
        }

        Self { peripheral: p }
    }
}

impl Drop for PeripheralGuard {
    fn drop(&mut self) {
        PeripheralClockControl::disable(self.peripheral);
    }
}

// We also have a zero-sized variant, because we can save some memory in a few cases
pub(crate) struct GenericPeripheralGuard<const P: u8> {}
```

These objects, together with `PeripheralClockControl` which contains the actual logic, implement
a reference counting scheme, so that peripherals with multiple channels aren't shut down when one
of the guards is dropped. LCD_CAM, for example, has a mostly independent half that provides
interfaces for cameras, besides the LCD interface.

This is all well and good, but we see the blocking driver working, so the peripheral must have a
clock signal, right? Right?

Let's open `esp-hal/src/lcd_cam/lcd/i8080.rs` and see! Looking at the driver [struct][6], we see
the following:

```rust
pub struct I8080<'d, Dm: DriverMode> {
    lcd_cam: PeripheralRef<'d, LCD_CAM>,
    tx_channel: ChannelTx<'d, Blocking, PeripheralTxChannel<LCD_CAM>>,
    _mode: PhantomData<Dm>,
}

impl<'d, Dm> I8080<'d, Dm>
where
    Dm: DriverMode,
{
    /// Creates a new instance of the I8080 LCD interface.
    pub fn new<P, CH>(
        lcd: Lcd<'d, Dm>,
        channel: impl Peripheral<P = CH> + 'd,
        mut pins: P,
        config: Config,
    ) -> Result<Self, ConfigError>
    where
        CH: TxChannelFor<LCD_CAM>,
        P: TxPins,
    {
        let tx_channel = ChannelTx::new(channel.map(|ch| ch.degrade()));

        let mut this = Self {
            lcd_cam: lcd.lcd_cam,
            tx_channel,
            _mode: PhantomData,
        };

        this.apply_config(&config)?;
        pins.configure();

        Ok(this)
    }
}
```

The driver stores the reference to the peripheral (so that it can't be used to create another,
conflicting driver), it stores the DMA channel it uses to transfer data and the `DriverMode`
(whether it is blocking or `async`). But apparently we forgot to store the `PeripheralGuard`.
Ouch. And indeed, [properly extracting and storing][7] the peripheral guard from `Lcd`, our
problem is solved.

However, this changes the mistery somewhat: if we didn't enable clocks for the driver, why did the
esp-hal example work?

The solution lies in the fact that LCD_CAM is two peripherals bundled as one. Both halves can hold
the clock enabled, and usually both need to be dropped for it to shut down. We saw that LCD did not
do its part here, so maybe the camera did?

In order to create a display driver, users first need to construct an `LcdCam`. This object is
used to configure the peripheral's `async`ness, because we don't have a good system in place to
allow mixing a blocking camera with an `async` LCD driver. Looking at the [constructor][8], we can
see that it creates guards for both halves:

```rust
impl<'d> LcdCam<'d, Blocking> {
    /// Creates a new `LcdCam` instance.
    pub fn new(lcd_cam: impl Peripheral<P = LCD_CAM> + 'd) -> Self {
        crate::into_ref!(lcd_cam);

        let lcd_guard = GenericPeripheralGuard::new();
        let cam_guard = GenericPeripheralGuard::new();

        Self {
            lcd: Lcd {
                lcd_cam: unsafe { lcd_cam.clone_unchecked() },
                _mode: PhantomData,
                _guard: lcd_guard,
            },
            cam: Cam {
                lcd_cam,
                _guard: cam_guard,
            },
        }
    }
}
```

The issue reproducer created the driver like this:

```rust
let hub75_peripherals = Hub75Peripherals {
    lcd_cam: peripherals.LCD_CAM,
    // ...
};

//...
type Hub75Type = Hub75<'static, esp_hal::Async>;
let mut hub75 =
    Hub75Type::new_async(hub75_peripherals.lcd_cam, pins, channel, tx_descriptors, 20.MHz())
        .expect("failed to create Hub75!");
//...

impl<'d> Hub75<'d, esp_hal::Async> {
    pub fn new_async<CH>(
        lcd_cam: LCD_CAM,
        hub75_pins: Hub75Pins,
        channel: impl Peripheral<P = CH> + 'd,
        tx_descriptors: &'static mut [DmaDescriptor],
        frequency: HertzU32,
    ) -> Result<Self, Hub75Error>
    where
        CH: TxChannelFor<LCD_CAM>,
    {
        let lcd_cam = LcdCam::new(lcd_cam).into_async();
        Self::new_internal(lcd_cam, hub75_pins, channel, tx_descriptors, frequency)
    }

    fn new_internal<CH>(
        lcd_cam: LcdCam<'d, DM>,
        hub75_pins: Hub75Pins,
        channel: impl Peripheral<P = CH> + 'd,
        tx_descriptors: &'static mut [DmaDescriptor],
        frequency: HertzU32,
    ) -> Result<Self, Hub75Error>
    where
        CH: TxChannelFor<LCD_CAM>,
    {
        // ...

        let i8080 = I8080::new(
            lcd_cam.lcd, // !!!
            channel,
            pins,
            i8080::Config {
                frequency,
                ..Default::default()
            },
        )
        .map_err(Hub75Error::I8080)?
        .with_ctrl_pins(NoPin, hub75_pins.clock);
        Ok(Self {
            i8080,
            tx_descriptors,
        })
    }
}
```

Everything boils down to `Hub75::new_internal`, which takes both halves of the `LcdCam` struct,
extracts the LCD part, and drops the rest. Since the `I8080` driver did not hold on to its clocks,
dropping the other parts ended up disabling the peripheral.

Comparing this to our blocking example, we can see the driver being initialized directly in the
example's `main` function:

```rust
//...
let lcd_cam = LcdCam::new(peripherals.LCD_CAM);
let mut i8080_config = Config::default();
i8080_config.frequency = 20.MHz();
let i8080 = I8080::new(lcd_cam.lcd, peripherals.DMA_CH0, tx_pins, i8080_config)
    .unwrap()
    .with_ctrl_pins(lcd_rs, lcd_wr);
//...
```

The example does not touch `lcd_cam` later. In Rust this means, that it is not dropped, consequently
the camera is not dropped, and the clocks are not disabled.

Mystery solved.

# Conclusion

In this post we have seen two separate issues, that lead to the same symptoms. In both cases the
issues are really obvious in hindsight, but actually finding the root cause took us on a bit of a
journey. The issues can be traced back to either an incorrect assumption, or a simple case of
inattention. We could have found the disabled clock quicker if esp-hal had more strategically placed
logging statements (which usually are placed only [after the fact][9]),
and maybe we could have saved a bit of time by using a proper debugger instead of the
add-logging-and-recompile cycle.

[1]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/dma/pdma/i2s.rs#L131-L137
[2]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/qa-test/src/bin/lcd_i8080.rs
[3]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/dma/gdma.rs#L221-L249
[4]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/dma/gdma.rs#L451-L479
[5]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/dma/gdma.rs#L620-L662
[6]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/lcd_cam/lcd/i8080.rs#L95-L99
[7]: https://github.com/esp-rs/esp-hal/pull/3007/files#diff-f06d944bf2db0ae80bd62724038f4beb4ee108971cc96e87457a4dfb5bfa7c7eR123
[8]: https://github.com/esp-rs/esp-hal/blob/53ba3ce95d6f3f89969539fe2ec3a3519773a368/esp-hal/src/lcd_cam/mod.rs#L36-L53
[9]: https://github.com/esp-rs/esp-hal/pull/3007/files#diff-a7e7ea78f78eadae91b41636c9ddfaf5907c74d60e406a90e6abafe904bbf71bR1076
