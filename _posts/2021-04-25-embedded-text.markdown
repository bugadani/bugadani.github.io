---
layout: post
title:  "Announcing embedded-text 0.4.1"
date:   2021-04-25 20:40 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new stable version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options and rich styling features.

This is a minor maintenance release that updates the `ansi-parser` dependency so that the `ansi`
feature can now be used in `no_std` environments.

To install, add the following to your `Cargo.toml` dependencies:

```toml
embedded-text = "0.4.1"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/0.4.1/embedded_text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.4.1/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.4.1/embedded_text/style/index.html
