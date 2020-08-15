---
layout: post
title:  "Announcing embedded-text 0.2.0"
date:   2020-08-15 13:50 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] is available for download. `embedded-text` extends
the excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options.

<!--more-->

The new major features of this release are:
 * Vertical text alignment
 * Support for some special characters:
   * zero-width space (`\u{200B}`)
   * nonbreaking space (`\u{A0}`)
   * carriage return (`\r`)

## Vertical text alignment

Using `TextBoxStyle` and `TextBoxStyleBuilder`, you can now set vertical text alignment to your
text boxes. The default alignment is `TopAligned`, and the complete list of supported vertical
alignments are:
 * `TopAligned`
 * `CenterAligned` (same type as used for horizontal centering)
 * `BottomAligned`

Additionally, all the alignment types can be imported by the prelude.

### Examples

```rust
let center_aligned = TextBoxStyle::new(Font6x8, BinaryColor::On, LeftAligned, CenterAligned);
let center_aligned = TextBoxStyleBuilder::new(Font6x8)
    .text_color(BinaryColor::On)
    .alignment(LeftAligned)
    .vertical_alignment(CenterAligned);
```

## New special characters

The new supported characters allow you to better control what is drawn to your display and how.
You can use `\r` to overwrite text, `\u{200B}` to add points to longer words where they may be
wrapped and `\u{A0}` to add a whitespace that prevents line breaks.

For a complete list of changes (excluding under the hood changes), see the [changelog]

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.2.0"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/bugadani/embedded-text
[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/
[changelog]: https://github.com/bugadani/embedded-text/blob/v0.2.0/CHANGELOG.md
