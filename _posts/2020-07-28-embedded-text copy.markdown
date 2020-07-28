---
layout: default
title:  "Announcing embedded-text 0.0.3"
date:   2020-07-28 15:21 +0200
categories: embedded-graphics embedded-text
---

A new test version of [`embedded-text`] is available for download. This version contains a few new
features, a lot of internal improvements and fixes identified issues.

The main new features:
 - Added a prelude to include common types.
 - Added new associated function `measure_text` to font objects. This function returns the height
   of the given piece of text when drawn into a given width.

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-text = "0.0.3"
```

For documentation, see [docs.rs]

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/bugadani/embedded-text
[docs.rs]: https://docs.rs/embedded-text/
