---
layout: post
title:  "Announcing embedded-text 0.3.0"
date:   2020-10-02 15:20 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] is available for download. `embedded-text` extends
the excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options.

<!--more-->

The new major features of this release are:
 * Height modes and Vertical overflow options
 * Support for new special characters:
   * soft hyphen (`\u{Ad}`)

## Height modes and Vertical overflow options

The following options are now available through the `height_mode` style option:
 - `Exact(OverflowMode)`: The `TextBox` will be rendered the exact size it was specified with.
 - `FitToText`: The `TextBox` will take up exactly as much vertical space as necessary. This option
   keeps the width and ignores the height parameter given to the `TextBox`'s constructor.
   You might have noticed that this mode does not have an `OverflowMode`. This is, because the
   `TextBox` always takes up as much space as necessary, so the text can't overflow.
 - `ShrinkToText(OverflowMode)`: If the text does not fit into the `TextBox`, this mode behaves the
   same as `Exact`. Otherwise, if the text does not take up all available space, the `TextBox` will
   shrink to the size of the text, just like `FitToText` would.

For the modes where you can specify an `OverflowMode`, you have the following options:
 - `Visible`: All the text will be displayed, regardless of the bounding box.
 - `FullRowsOnly`: Text will only be drawn inside the bounding box. Partially drawn lines are
   not rendered.
 - `Hidden`: Text will only be drawn inside the bounding box. Partially drawn lines are rendered.

### Example configuration

```rust
let style = TextBoxStyleBuilder::new(Font6x8)
    .text_color(BinaryColor::On)
    .alignment(LeftAligned)
    .vertical_alignment(CenterAligned)
    .height_mode(ShrinkToText(Hidden))
    .build();
```

## New special characters

The new supported characters allow you to better control what is drawn to your display and how.
You can use the `\u{ad}` (soft hyphen) character to specify position when words can be wrapped if
they don't fit the current line. When those words are broken, the soft hyphen character will be
displayed at the end (or beginning, depending on the space available) of the line.

For a complete list of changes (excluding some under the hood changes), see the [changelog]

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.3.0"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/bugadani/embedded-text
[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/
[changelog]: https://github.com/bugadani/embedded-text/blob/v0.3.0/CHANGELOG.md
