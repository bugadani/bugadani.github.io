---
layout: post
title:  "Announcing embedded-text 0.4.0"
date:   2020-11-26 15:30 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] (which is now part of the embedded-graphics organization 🎉)
is available for download. `embedded-text` extends the excellent [`embedded-graphics`] library with
a `TextBox` object that supports multiline text rendering with the common horizontal alignment options.

<!--more-->

The new major features of this release are:
 * In-line styling using ANSI escape sequences
 * `Scrolling` vertical alignment
 * New text style options:
   * Underlined (via `TextBoxStyleBuilder::underlined()`)
   * Strike-through (via `TextBoxStyleBuilder::strikethrough()`)
 * Support for new special characters:
   * tab (`\t`) with configurable tab size

## Partial ANSI escape sequence support

You can now use ANSI escape sequences to style your text. ANSI sequences provide a method to define
text style in line with the text itself. This is a more flexible, but perhaps a bit more involved
method to style your text compared to specifying a static style using `TextBoxStyleBuilder`.
For the list of supported sequences, please see the [documentation][ansi-docs].
Check out the `colored_text` example for a demo, which renders the following image:

![text styled with ANSI escape sequences](/assets/colored_text.png)

## Scrolling alignment

Scrolling alignment is a combination of the Top and Bottom vertical alignments. If the text does not
completely fill the bounding box, it behaves as Top aligned, otherwise as Bottom aligned. This is
especially suited for terminal-like, editor-like applications where the last line should always be
visible.
See the `editor` example for a demo application.

## New special characters

You can now use (horizontal) tab characters `\t` to tabulate text. The default tab width is 4 spaces
and it is configurable through `TextBoxStyleBuilder::tab_size()`. This function takes a `TabSize`
struct, which can represent either a number of pixels or a number of space characters.
See the `tabs` example for how to configure tab size.

For a complete list of changes (excluding some under the hood changes), see the [changelog]

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.4.0"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.4.0/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.4.0/embedded_text/style/index.html
