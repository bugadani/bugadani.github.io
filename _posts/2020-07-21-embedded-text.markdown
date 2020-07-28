---
layout: default
title:  "Announcing embedded-text 0.0.1"
date:   2020-07-21 15:50 +0200
categories: embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

The first test version of [`embedded-text`] is available for download. [`embedded-text`]
implements a `TextBox` struct with similar API as [`embedded-graphics`] `Text`, supporting multiline
text rendering in a bounding box. `embedded-text` supports the common horizontal text alignment
options:

 * `LeftAligned`
 * `RightAligned`
 * `CenterAligned`
 * `Justified`

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.0.1"
```

<!--more-->

There are still quite a lot to do before `embedded-text` could be considered stable.

For documentation, see [docs.rs]

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[`embedded-text`]: https://github.com/bugadani/embedded-text
[docs.rs]: https://docs.rs/embedded-text/
