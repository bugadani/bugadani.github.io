---
layout: post
title:  "Announcing embedded-text 0.5.0"
date:   2021-09-08 18:15 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common text alignment options, rich styling features and more!

`0.5.0` is a major update over the previous (`0.4.x`) iteration including new features, significant
usability changes and bugfixes. `0.5.0` is also the first stable release built on top of
`embedded-graphics` version `0.7`.

## New features

### Plugins

The new embedded-text now support plugins! Plugins are an experimental feature to extend the 
capabilities of the crate. To implement a plugin, users need to enable the `plugins` cargo feature.

Note that imlementing a new plugin is experimental and the API may break without prior notice.

The crate provides a small number of plugins that can be used without enabling the `plugins`
feature.

 * `Tail`

    The `Tail` plugin can be used to keep the end of the text (the tail) in the displayed area.
    Example: in the left column, the text is top aligned. In the right column, the same text is
    displayed top-aligned, but with the `Tail` plugin active.

    ![Image showing the Tail plugin in action](/assets/0.5.0/tail.png)

    ```rust
    use embedded_text::{
      plugin::tail::Tail, TextBox, ...
    };

    TextBox::new(...)
      .add_plugin(Tail) // < where the magic happens
      .draw(&mut display)?;
    ```

 * `Ansi`

    Enables text styling using (some) [Ansi sequences]. While this feature has been part of previous
    versions, now you can add it to select `TextBox` instances instead of enabling globally.

    ![Ansi sequence demo showcasing advanced styling](/assets/colored_text.png)

    ```rust
    use embedded_text::{
      plugin::ansi::Ansi, TextBox, ...
    };
    
    TextBox::new(...)
      .add_plugin(Ansi::new()) // < where the magic happens
      .draw(&mut display)?;
    ```

[Ansi sequences]: https://en.wikipedia.org/wiki/ANSI_escape_code

### Other, smaller features

 * Vertical text offsetting

    The vertical position of the text within a text box can now be modified using the
    `vertical_offset` text box option. Negative values move the text up, while positive values move
    it down.

    `vertical_offset` is a field of `TextBox`.

    ```rust
    use embedded_text::TextBox;
    ...
    let mut text_box = TextBox::new(...);
    text_box.vertical_offset = 10; // move text down by 10 pixels
    ```

 * Paragraph spacing

    The vertical distance between paragraphs (sections of text separated by a newline `\n` character)
    can now be changed using the `paragrap_spacing` style option.

    ```rust
    use embedded_text::style::TextBoxStyleBuilder;
    ...
    let textbox_style = TextBoxStyleBuilder::new()
      ...
      .paragraph_spacing(6)
      .build();
    ```

    In the following example picture, the paragraph spacing of the left text box is 0, while the
    right is 6.

    ![Two columns of text with different paragraph spacing](/assets/0.5.0/paragraph_spacing.png)

 * Configurable space rendering

    You can now force to render or hide (collapse) the leading or trailing spaces in lines. This
    configuration does not change how the lines are measured and wrapped, it only affets what is
    rendered.

    Note that if a line is wrapped at a space (in case the space does not fit in the current line),
    a single space is consumed and will not be displayed or carried over to the next line.

    ![One column of text that renders leading spaces, and one column that renders trailing spaces](/assets/0.5.0/whitespace_control.png)

## Usability changes

These changes enable users to use embedded-text in a more flexible or simple way.

### `TextBox` configuration options are no longer encoded in the type of the text box object.

Previously, style options like alignment were encoded in the type of the text box. This meant that
storing the text box object in a variable (e.g. a struct field in an UI) was cumbersome and did not
allow much flexibility.

Starting with `0.5.0`, embedded-text uses non-zero sized values (e.g. enums) as configuration,
removing the old myriad of type parameters, making embedded-text easier to use and simpler to
configure.

### New constructor functions

To enable simpler construction of objects and to bring the API closer to embedded-graphics, this
release adds new constructors to `TextBox` and `TextBoxStyle`. Use these functions if you only
want to change a single property from the default style. These new constructors are:

 * `TextBox::with_alignment` / `TextBoxStyle::with_alignment`
 * `TextBox::with_vertical_alignment` / `TextBoxStyle::with_vertical_alignment`
 * `TextBox::with_height_mode` / `TextBoxStyle::with_height_mode`
 * `TextBox::with_line_height` / `TextBoxStyle::with_line_height`
 * `TextBox::with_paragraph_spacing` / `TextBoxStyle::with_paragraph_spacing`
 * `TextBox::with_tab_size` / `TextBoxStyle::with_tab_size`
 * `TextBox::with_vertical_offset`

## Removed 

This list is not exhaustive.

 *  A substantial amount of the API (internal or not intended as public) has been hidden to reduce
    clutter.
 * `TextBoxStyle` objects can no longer be constructed (it is now `#[non_exhaustive]`).
 * `Scrolling` vertical alignment.
 * Ansi sequence support has been removed from the base library and reimplemented as the `Ansi`
   plugin.

For a complete list of changes (excluding some under the hood changes), see the [changelog].

To install, add the following to your `Cargo.toml` dependencies:

```toml
embedded-text = "0.5.0"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/0.5.0/embedded_text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.5.0/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.5.0/embedded_text/style/index.html
