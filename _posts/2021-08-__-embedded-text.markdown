---
layout: post
title:  "Announcing embedded-text 0.5.0"
date:   2021-08-?? ??:?? +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common text alignment options, rich styling features and more!

`0.5.0` is a major update over the previous (`0.4.x`) iteration including new features, major
usability changes and bugfixes. `0.5.0` is also the first stable release built on top of
`embedded-graphics` version `0.7`.

## New features

 * plugins
 * vertical offsetting
 * paragraph spacing
 * configurable space rendering

## Usability changes

 * `TextBox` configuration options are no longer encoded in the type of the text box object.
 * constructors

## Removed 

 * `TextBoxStyle` objects can no longer be constructed (it is now `#[non_exhaustive]`).
 * `Scrolling` vertical alignment.

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
