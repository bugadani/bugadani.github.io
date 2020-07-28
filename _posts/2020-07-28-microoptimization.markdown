---
layout: default
title:  "Micro-optimization of the day: bit masks"
date:   2020-07-28 17:53 +0200
categories: rust optimization
excerpt_separator: <!--more-->
---

In an embedded environment, using bit masks is quite common. Getting the `nth` bit of a word isn't
particularly difficult, a bitshift and a binary and does the trick:

```rust
pub fn bit_n(bit: u32) -> u32 {
    1 << bit % 32
}
```

Nice and simple. Sometimes, however, we need to get a bit from the other end. To do this, we
have two possibilities:
 - get the `width - nth` bit using something like above
 - construct our pattern some different way, i.e. by right shifting the leftmost bit

```rust
pub fn left_bit_n_1(bit: u32) -> u32 {
    1 << 31 - bit % 32
}

pub fn left_bit_n_2(bit: u32) -> u32 {
    0x8000_0000 >> bit % 32
}
```

What can we tell about their performance?

<!--more-->

Due to how common and simple this problem is, I fully expected the Rust compiler
(rustc 1.47.0-nightly (6c8927b0c 2020-07-26)) to optimize these functions to the same code,
however when examining the generated code, I was greeted with a
more-or-less straight translation of the source code:

Compiled for ARM (`--target=thumbv7em-none-eabihf`):

```asm
example::left_bit_n_1:
  movs r1, #31
  bic.w r0, r1, r0
  movs r1, #1
  lsl.w r0, r1, r0
  bx lr

example::left_bit_n_2:
  and r0, r0, #31
  mov.w r1, #-2147483648
  lsr.w r0, r1, r0
  bx lr
```

Or compiled for x86_64 (`--target=x86_64-pc-windows-msvc`):

```asm
example::left_bit_n_1:
  not cl
  mov eax, 1
  shl eax, cl
  ret

example::left_bit_n_2:
  mov eax, -2147483648
  shr eax, cl
  ret
```

All right, the second function is shorter, so it's faster, right? *Maybe.*

On an ARM Cortext-M4 MCU, we can be confident (but not *sure* without actually measuring it) that
the shorter code performs better.

This topic gets a lot more complicated if we are looking at desktop computers. Today's CPUs are
*extremely* complex and employ their own optimizations that can be hard to predict, so in order to
figure out if our shorter function is actually better, we need to **measure** it.

Thankfully, using a crate like [Criterion.rs] this is simple enough:

```rust
use example::*;
use criterion::*;

fn bench_left_shift(c: &mut Criterion) {
    c.bench_function("left shift", |b| b.iter(|| left_bit_n_1(black_box(20))));
}

fn bench_right_shift(c: &mut Criterion) {
    c.bench_function("right shift", |b| b.iter(|| left_bit_n_2(black_box(20))));
}

criterion_group!(basics, bench_left_shift, bench_right_shift);
criterion_main!(basics);
```

The results:

```
left shift              time:   [1.2492 ns 1.2555 ns 1.2628 ns]
right shift             time:   [1.2478 ns 1.2515 ns 1.2558 ns]
```

When benchmarked on x86_64 (a Ryzen R5 3600 CPU), there is practically no difference in execution time.
This isn't exactly what I expected, but there are possibilities why this happens:

 - either the function's results are cached somewhere, somehow
 - or the two different sets of instructions result in microcode of the same complexity and performance

We can rule out the first option by providing random input. Right?

```rust
use example::*;
use criterion::*;
use rand::Rng;

fn bench_left_shift(c: &mut Criterion) {
    let mut rng = rand::thread_rng();
    c.bench_function("left shift", |b| {
        b.iter(|| left_bit_n_1(black_box(rng.gen())))
    });
}

fn bench_right_shift(c: &mut Criterion) {
    let mut rng = rand::thread_rng();
    c.bench_function("right shift", |b| {
        b.iter(|| left_bit_n_2(black_box(rng.gen())))
    });
}

criterion_group!(basics, bench_left_shift, bench_right_shift);
criterion_main!(basics);
```

The results (ignore the overall higher times, that's the RNG):

```
left shift              time:   [3.9048 ns 3.9142 ns 3.9249 ns]
right shift             time:   [3.8884 ns 3.9008 ns 3.9154 ns]
```

No difference at all.

Conclusion:
-----------

 * The optimized code produced by Rust and LLVM is good, but not perfect - don't expect fundamental
   changes in your implemented algorithms, even in the simplest cases.
 * If you suspect your code can be made faster, inspect what's generated, and actually
   _measure_ it's performance.
 * Shorter machine code is usually better, but not necessarily faster.
 * Code like this is rarely the performance bottleneck. While it's not bad to work with the function
   that results in less machine code, it may not be worth putting in hours and hours trying to
   figure out how to optimize the last possible instruction. It can be fun, though 😅.

**Important note:** Simple benchmarks like this may work well in a vacuum, but their results may
 not translate well into real world performance difference. Bigger programs, especially when
 executed on a modern CPU can have performance characteristics that may be seemingly unexpected.

 It's possible that less code results in better cache alignment, and can be faster when executed in
 a more complex environment. But it's also possible that less code actually results in worse overall
 cache alignment.

[Criterion.rs]: https://crates.io/crates/criterion
