---
layout: default
title:  "Improving embedded-graphics' draw performance"
date:   2020-06-17 08:25 +0200
categories: embedded-graphics ssd1306 performance
excerpt_separator: <!--more-->
---

I'm always looking for ways to squeeze the most performance out of particularly hot code. [Embedded-graphics](https://github.com/jamwaffles/embedded-graphics) and the [SSD1306](https://github.com/jamwaffles/ssd1306) driver gives some opportunity to do this, while also allowing me to extend e-g with a [feature](https://github.com/jamwaffles/embedded-graphics/issues/339) I need.

<!--more-->

[`GraphicsMode::set_pixel()`](https://github.com/jamwaffles/ssd1306/blob/f5aa1d7b8f6b0ebb695fca312c3d999e52ecdb74/src/mode/graphics.rs#L176) has several functions besides writing to the framebuffer:
 - Transforms coordinates based on image rotation
 - Prevents drawing outside of the display
 - Keeps track of which part of the display was updated

   > This is a nice feature as it allows only transmitting data that actually changed, which can greatly improve performance when using slower display connections.

 - Works with several display sizes!

The way it's implemented, the driver check **for every pixel it draws** which display size to use,
should rotation be applied, and whether the draw is inside the framebuffer. Also, Rust checks whether or not calculating the byte index overflows.

 > Don't get me wrong. The driver is awesome and these things are totally fine for most applications.

 > This is not a tutorial, so I won't explain everything. This is just a basic idea and some code for demonstration. Adapt it on your own risk.

# Phase 1: copy & cleanup

The way the SSD1306 driver is implemented allows us to roll our own version of `GraphicsMode`. For simplicity's sake, I only need one display size and no rotation, so I'm not implementing anything to handle those, but there [are](https://github.com/jamwaffles/ssd1306/pull/125) [ways](https://github.com/jamwaffles/ssd1306/pull/130) to support them efficiently.

So what I did was to copy `GraphicsMode`, delete some code and end up with this:

```rust
pub struct FastGraphicsMode<DI> {
    properties: DisplayProperties<DI>,
    buffer: [u8; 1024],
}

impl<DI> DisplayModeTrait<DI> for FastGraphicsMode<DI> {
    fn new(properties: DisplayProperties<DI>) -> Self {
        FastGraphicsMode {
            properties,
            buffer: [0; 1024],
        }
    }

    fn into_properties(self) -> DisplayProperties<DI> {
        self.properties
    }
}

impl<DI> FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    pub fn clear(&mut self) {
        self.buffer = [0; 1024];
    }

    pub fn flush(&mut self) -> Result<(), DisplayError> {
        self.properties.bounded_draw(
            &self.buffer,
            128,
            (0, 0),
            (128, 63),
        )
    }

    // Note that this is function assumes you know what you do so make sure
    // You are only drawing to the valid area on the display.
    fn modify_pixel(&mut self, x: u32, y: u32, op: impl FnOnce(u8, &mut u8)) {
        unsafe { core::intrinsics::assume(x < 128); }
        unsafe { core::intrinsics::assume(y < 64); }

        let idx = ((y & 0xFFFFFFF8) * 16 + x) as usize;
        let bit: u8 = (y % 8) as u8;

        op(bit, unsafe {self.buffer.get_unchecked_mut(idx) });
    }

    pub fn write_pixel(&mut self, x: u32, y: u32, value: u8) {
        self.modify_pixel(x, y, |bit, byte| {
            *byte = *byte & !(1 << bit) | (value << bit);
        })
    }

    pub fn init(&mut self) -> Result<(), DisplayError> {
        self.clear();
        self.properties.init_column_mode()?;

        self.properties.set_draw_area(
            (0, 0),
            (128, 63),
        )
    }

    pub fn set_brightness(&mut self, brightness: Brightness) -> Result<(), DisplayError> {
        self.properties.set_brightness(brightness)
    }
}

impl<DI> DrawTarget<BinaryColor> for FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: drawable::Pixel<BinaryColor>) -> Result<(), Self::Error> {
        let drawable::Pixel(pos, color) = pixel;

        self.write_pixel(pos.x as u32, pos.y as u32, RawU1::from(color).into_inner());

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(128, 64)
    }
}
```

That's not so bad, but we can take it a step further. Notice the `RawU1::from(color).into_inner()` call in `draw_pixel` and `*byte = *byte & !(1 << bit) | (value << bit);` in `write_pixel`. We don't need to first clear a pixel, then write it. Either we want to clear a bit, or to set it. So that's next.

 > `write_pixel` and `modify_pixel` are not part of the SSD1306 driver, but a rewrite of `set_pixel` to allow code reuse later.

# Phase 2: Introduce our own PixelColor

Enter: Rust's Unit-like structures. What are those? [The book](https://doc.rust-lang.org/book/ch05-01-defining-structs.html#unit-like-structs-without-any-fields) is surprisingly taciturn here, all we get are these 3 lines:

 > You can also define structs that don’t have any fields! These are called unit-like structs because they behave similarly to (), the unit type. Unit-like structs can be useful in situations in which you need to implement a trait on some type but don’t have any data that you want to store in the type itself.

Okay, so we have types that don't have any data. How are those useful?

Consider the above code:

```rust
impl<DI> DrawTarget<BinaryColor> for FastGraphicsMode<DI>
```

This implements `DrawTarget` so that functions that rely on it, can use our new graphics mode. But it only implements `DrawTarget` when the embedded-graphics draw operations are using `BinaryColor`.

The idea is to replace `BinaryColor` with our own color type(s) so that we can provide specialized code for each color we want to support. Unit-like structures help to achieve this.

Our new colors look something like this:

```rust
#[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash, Debug)]
pub struct PixelOn;

#[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash, Debug)]
pub struct PixelOff;

impl PixelColor for PixelOn {
    type Raw = RawU1; // don't worry about this
}

impl PixelColor for PixelOff {
    type Raw = RawU1; // don't worry about this
}
```

We also have to tell our code how to render a particular color:

```rust
impl<DI> FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    // ...

    pub fn set_pixel(&mut self, x: u32, y: u32) {
        self.modify_pixel(x, y, |bit, byte| {
            *byte = *byte | (1 << bit);
        });
    }

    pub fn clear_pixel(&mut self, x: u32, y: u32) {
        self.modify_pixel(x, y, |bit, byte| {
            *byte = *byte & !(1 << bit);
        });
    }

    // ...
}

impl<DI> DrawTarget<PixelOn> for FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: drawable::Pixel<PixelOn>) -> Result<(), Self::Error> {
        let drawable::Pixel(pos, _) = pixel;

        // ... which makes the `as` coercions here safe.
        self.set_pixel(pos.x as u32, pos.y as u32);

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(128, 64)
    }
}

impl<DI> DrawTarget<PixelOff> for FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: drawable::Pixel<PixelOff>) -> Result<(), Self::Error> {
        let drawable::Pixel(pos, _) = pixel;

        // ... which makes the `as` coercions here safe.
        self.clear_pixel(pos.x as u32, pos.y as u32);

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(128, 64)
    }
}
```

This seems like a lot of duplicated code, and it is. But the result is that the compiler generates completely specialized code for the different colors, making our replacement of `set_pixel` as fast as possible. This duplication also makes our next step a lot simpler.

# Phase 3: InvertPixel

Imagine a progress bar with some text drawn on top of it. With only fixed colors, it would be rather complicated to implement:
 - Draw the text
 - Draw the progress bar background
 - Some of the next is now overwritten, so let's redraw only that part with the inverse color

The hard part is drawing the text the second time, because we would need to be able to partially draw a character. We could switch some of the steps, but we would still need partial character draw. Ouch.

However, if we had an `InvertPixel` "color", the process would be a lot less complicated:
 - Draw progress bar background
 - Using the inverting color, draw the text

Faster, simpler, and a lot less prone to subtle bugs.

To support this, we only need to add the following:

```rust
#[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash, Debug)]
pub struct InvertPixel;

impl PixelColor for InvertPixel {
    type Raw = RawU1; // to be fair, this is a lie, we can't represent 3 colors on 1 bit, but we'll not use this at all, so it's fine
}

impl<DI> FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    // ...

    pub fn invert_pixel(&mut self, x: u32, y: u32) {
        self.modify_pixel(x, y, |bit, byte| {
            *byte = *byte ^ (1 << bit);
        });
    }

    // ...
}

impl<DI> DrawTarget<InvertPixel> for FastGraphicsMode<DI>
where
    DI: WriteOnlyDataCommand,
{
    type Error = DisplayError;

    fn draw_pixel(&mut self, pixel: drawable::Pixel<InvertPixel>) -> Result<(), Self::Error> {
        let drawable::Pixel(pos, _) = pixel;

        // ... which makes the `as` coercions here safe.
        self.invert_pixel(pos.x as u32, pos.y as u32);

        Ok(())
    }

    fn size(&self) -> Size {
        Size::new(128, 64)
    }
}
```

That wasn't so bad for a feature like this. In fact, we didn't need to touch `embedded-graphics` at all, which is surprising and awesome!

# The ToDo

 - To support the full featureset of `embedded-graphics`, we need to implement `From` for our colors. I don't need that now, so I'll stop here.
 - Benchmarking - I can see some improvements in my application's binary size, and in runtime performance. How much? I need to measure it some day.

# Conclusions

Rust is awesome.