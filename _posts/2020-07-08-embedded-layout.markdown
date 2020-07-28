---
layout: post
title:  "Announcing embedded-layout 0.1.0"
date:   2020-07-08 9:12 +0200
categories: embedded-graphics embedded-layout
excerpt_separator: <!--more-->
---

The first stable version of [`embedded-layout`] is available for download. [`embedded-layout`]
extends [`embedded-graphics`] with tools to align objects to each other and make it easier to compose
complex screen layouts more easily.

To install, add the following to your `Cargo.toml` dependencies:
```toml
embedded-layout = "0.1.0"
```

<!--more-->

This release marks the first stable release of [`embedded-layout`]. The biggest goal of this release
was to introduce `LinearLayout`, provide code examples and improve the documentation.

A `LinearLayout` has a primary **orientation** and a secondary **alignment**. Orientation
selects between top-to-bottom and left-to-right layout, while alignment works in the other direction,
i.e. to horizontally center views that are laid out vertically.
`LinearLayout` also supports different element spacings. These can be used to specify a margin
between the views, or to distribute views in a given space.
`LinearLayout` is implemented on top of the new `ViewGroup`. When a layout is finalized by calling
`arrange`, it returns a `ViewGroup`. A `ViewGroup` is a `View` object that can contain multiple
number of other `View` objects.

Compared to `0.0.1`, there are a number of other changes, like new alignments
(for example, `horizontal::LeftToRight`), and many more internal changes.

For more information, examples, and documentation, see the [`embedded-layout`] repository on GitHub
and the [docs.rs] page.

I really hope you give [`embedded-layout`] a try! If you have any questions, suggestions, issues,
feature requests, or if you'd like to contribute, feel free to open an issue or a pull request on
the GitHub repository!

[`embedded-graphics`]: https://github.com/jamwaffles/embedded-graphics
[`embedded-layout`]: https://github.com/bugadani/embedded-layout
[docs.rs]: https://docs.rs/embedded-layout/
