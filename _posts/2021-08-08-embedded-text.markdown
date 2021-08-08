---
layout: post
title:  "Announcing embedded-text 0.5.0-beta.3"
date:   2021-07-10 17:30 +0200
categories: rust embedded-graphics embedded-text
excerpt_separator: <!--more-->
---

A new beta version of [`embedded-text`] is available for download. `embedded-text` extends the
excellent [`embedded-graphics`] library with a `TextBox` object that supports multiline text
rendering with the common horizontal alignment options and rich styling features.

`0.5.0-beta.3` includes refinements to the plugin system, as well as stable plugins that can be
used without enabling the `plugin` feature.

## Stable plugins

 * `Tail`: The `Tail` plugin replaces the `Scrolling` vertical alignment.
   Use this to display the last lines of your text. The new plugin allows you to use initial (i.e.
   when the text fits the text box) vertical alignments other than `Top`.
 * `Ansi`: ANSI sequence support has been moved from the core of the library into the `Ansi` plugin.
   You can still disable the `ansi` feature if you want to reduce compile time by removing unused
   dependencies.

To use a plugin, call the `TextBox::add_plugin()` method. Starting with `beta.3`, multiple plugins
can be used at the same time.

*Note*: Plugin support is currently experimental and unrefined. Expect breaking changes, or even
complete removal in the future.

For a complete list of changes (excluding some under the hood changes), see the [changelog],

To install, add the following to your `Cargo.toml` dependencies:

```toml
embedded-text = "0.5.0-beta.3"
```

For documentation, see [docs.rs].

I really hope you give [`embedded-text`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-text`]: https://github.com/embedded-graphics/embedded-text
[`embedded-graphics`]: https://github.com/embedded-graphics/embedded-graphics
[docs.rs]: https://docs.rs/embedded-text/0.5.0-beta.3/embedded_text/
[changelog]: https://github.com/embedded-graphics/embedded-text/blob/v0.5.0-beta.3/CHANGELOG.md
[ansi-docs]: https://docs.rs/embedded-text/0.5.0-beta.3/embedded_text/style/index.html
