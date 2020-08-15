---
layout: post
title:  "Announcing embedded-text 0.1.0"
date:   2020-07-31 20:33 +0200
categories: embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

The first stable version of [`embedded-text`] is available for download. `embedded-text` extends
the excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options.

<!--more-->

This version is meant to stabilize `embedded-text` and contains several bugfixes and code quality
improvements.

Highlights:

 - `TextBoxStyle::from_text_style` to create `TextBox` style from embedded-graphics' `TextStyle` objects.
 - Moved `FontExt::measure_text` to `TextBoxStyle::measure_text_height`.

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.1.0"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/bugadani/embedded-text
[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/
