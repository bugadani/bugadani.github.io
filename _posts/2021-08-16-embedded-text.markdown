---
layout: post
title:  "Announcing embedded-text 0.5.0-beta.4"
date:   2021-08-16 22:30 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new beta version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options and rich styling features.

`0.5.0-beta.4` is a minor update which includes a bunch of bug fixes and two new style properties
to control leading and trailing space rendering, which might be interesting for certain plugins, or
when rendering text with background color.

For a complete list of changes (excluding some under the hood changes), see the [changelog].

To install, add the following to your `Cargo.toml` dependencies:

```toml
embedded-text = "0.5.0-beta.4"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/0.5.0-beta.4/embedded_text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.5.0-beta.4/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.5.0-beta.4/embedded_text/style/index.html
