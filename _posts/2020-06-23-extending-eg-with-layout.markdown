---
layout: post
title:  "Extending embedded-graphics with simple layout functions"
date:   2020-06-23 09:19 +0200
categories: embedded-graphics features
excerpt_separator: <!--more-->
---

The first version of [`embedded-layout`] is available for download. `embedded-layout` aims at making [`embedded-graphics`] easier to use, by allowing users to align elements to each other.

<!--more-->

I've often found myself doing the same task over and over again: calculating positions for UI elements in my embedded application. Most of these positions consist of centering something, or moving to a corner of a display. These operations require me to calculate the size of the object I'm positioning and there's always the possibility of making an off-by-one error in the process.

With `embedded-layout`, these alignment operations are as simple as setting three parameters: a reference rectangle to which to align, and a horizontal and a vertical alignment option. That's it.

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-layout = "0.0.1"
```

The obligatory "Hello, world!" example:

```rust
use embedded_layout::prelude::*;
use embedded_graphics::{
    prelude::*,
    fonts::{Font6x8, Text},
    geometry::Point,
    primitives::Rectangle,
    pixelcolor::BinaryColor,
    style::TextStyleBuilder,
};

// Assuming disp is our draw target, see embedded-graphics for details
let display_area = disp.display_area();

let text_style = TextStyleBuilder::new(Font6x8)
                        .text_color(BinaryColor::On)
                        .build();

Text::new("Hello, world!", Point::zero())
     .into_styled(text_style)
     .align_to(display_area, horizontal::Center, vertical::Center) // this is where the magic happens
     .draw(&mut disp)
     .unwrap();
```

The currently supported alignment options:
 - horizontal:
   * NoAlignment
   * Center
   * Left
   * Right
 - vertical:
   * NoAlignment
   * Center
   * Top
   * Bottom

# Adding your own alignment option

While the current set of alignment options is not final, you may want to add other options that may or may not be included in the package.

To do this, you need to implement either the `HorizontalAlignment` or the `VericalAlignment` traits.

For example, to create a horizontal indenting alignment, you might want to do something like the following:

```rust
use embedded_graphics::prelude::*;
use embedded_layout::{prelude::*, HorizontalAlignment};

/// Align the left side of the object to the left side of the reference,
/// indented by 4 pixels
pub struct Indent;

impl HorizontalAlignment for Indent {
    fn align(&self, what: &impl Dimensions, reference: &impl Dimensions) -> i32 {
        let left_align = horizontal::Left;
        left_align.align(what, reference) + 4
    }
}
```

I hope you give `embedded-layout` a try. If you have any issues or feedback, feel free to open an issue!

[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[`embedded-layout`]: https://github.com/bugadani/embedded-layout
