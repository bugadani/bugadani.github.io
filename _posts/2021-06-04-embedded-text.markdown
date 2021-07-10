---
layout: post
title:  "Announcing embedded-text 0.5.0-beta.1"
date:   2021-06-04 20:00 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new beta version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options and rich styling features.

This release is a major step in the evolution of `embedded-text`. It targets the next generation of
`embedded-graphics`. Over the last few months, an incredible amount of work has been done on text
support in `embedded-graphics` and this work will be the foundation of some of the future changes in
`embedded-text`.

The new features of `0.5.0-beta.1` include:

 - `embedded-text` now uses `embedded-graphics` 0.7, which allowed removing several workarounds as
   well as simplifying the codebase.
 - The typestate-based configuration options have been replaced by enum values. This means
   `embedded-text` can be used in a more dynamic way than before.
 - Vertical text offset
   The rendered text can be moved vertically, enabling scrolling effects.
 - Paragraph spacing
 - Various updates to the API to make it similar to `embedded-graphics`'s `Text` API.

For a complete list of changes (excluding some under the hood changes), see the [changelog],

To install, add the following to your `Cargo.toml` dependencies:

```toml
embedded-text = "0.5.0-beta.1"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/0.5.0-beta.1/embedded_text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.5.0-beta.1/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.5.0-beta.1/embedded_text/style/index.html
