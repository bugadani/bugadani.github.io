---
layout: post
title:  "Coerce the compiler to help us debug Axum handlers"
date:   2022-07-08 10:00 +0200
categories: rust axum
excerpt_separator: <!--more-->
---

Let's state the obvious first: [axum] is good, it's easy to use, it's fast and all that. But have
you ever seen an error like this?

```
❯ cargo c
    Checking axum-async v0.1.0 (C:\_Hobby\axum-async)
error[E0277]: the trait bound `fn() -> impl Future<Output = ()> {page}: Handler<_, _>` is not satisfied
   --> src\main.rs:21:44
    |
21  |     let app = Router::new().route("/", get(page));
    |                                        --- ^^^^ the trait `Handler<_, _>` is not implemented for `fn() -> impl Future<Output = ()> {page}`
    |                                        |
    |                                        required by a bound introduced by this call
    |
    = help: the trait `Handler<T, ReqBody>` is implemented for `Layered<S, T>`
note: required by a bound in `axum::routing::get`
   --> C:\Users\bugad\.cargo\registry\src\github.com-1ecc6299db9ec823\axum-0.5.11\src\routing\method_routing.rs:396:1
    |
396 | top_level_handler_fn!(get, GET);
    | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `axum::routing::get`
    = note: this error originates in the macro `top_level_handler_fn` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
error: could not compile `axum-async` due to previous error
```

Rust is so good with its error messages that it's just striking how very unhelpful this one is.
Let's see what we can do here!

<!--more-->

Setup
-----

So, our initial example is nice and simple: we have a page that does some work, then sends a polite
response.

```rust
use std::{net::SocketAddr, time::Duration};

use axum::{routing::get, Router};
use tokio::time::sleep;

async fn page() -> &'static str {
    sleep(Duration::from_secs(1)).await;
    "Hello one second later"
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(page));

    let addr = SocketAddr::from(([0, 0, 0, 0], 80));
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

So far so good, this compiles and works just fine, as expected. Our difficulties emerge when we try
to do something a tiny bit more complicated:

```rust
// all our previous imports
use std::rc::Rc;

async fn page() -> &'static str {
    let rc = Rc::new(());
    sleep(Duration::from_secs(1)).await;
    "Hello one second later"
}

// all else remains the same
```

If we modify our example to use this `page` function, we are greeted with the above error. This is
especially confusing since we have only modified the implementation of our function, and not its
signature, but somehow its type changed regardless.

Why is this?
------------

Axum requires that the page handlers are `Send`. Not the `async` functions themselves, but the
`Future` that is created when they are called. The compiler [tries its best][Send approximation] to
determine whether an `async fn`'s `Future` is `Send`. This is the process that bites into our ankles
here - if we use a non-`Send` type in the implementation, and that object "is held across an
`.await` point", then our `Future` is no longer `Send`. In other words, our previous `page` is not
`Send`, but the following is:

```rust
async fn send_page() -> &'static str {
    {
        let rc = Rc::new(());
        // We drop `rc` here so it's not "held across an .await point".
    }
    sleep(Duration::from_secs(1)).await;
    "Hello one second later"
}
```

Alright, but how do I find this in a complex program?
-----------------------------------------------------

If you've checked out the [Send approximation] page in the [Asynchronous Programming in Rust][async book],
you might wonder why our original error message is so unhelpful. That book states that the compiler
can produce **extremely** helpful error messages, like the following:

```
[spoiler content cut]
note: future is not `Send` as this value is used across an await
  --> src\main.rs:9:34
   |
7  |     let rc = std::rc::Rc::new(());
   |         -- has type `Rc<()>` which is not `Send`
8  | 
9  |     sleep(Duration::from_secs(1)).await;
   |                                  ^^^^^^ await occurs here, with `rc` maybe used later
10 | }
   | - `rc` is later dropped here
[spoiler content cut]
```

Unfortunately, axum does something that prevents helpful diagnostics like this one. But there is a 
way to wringle this information out of the compiler. We can construct a function that takes any
`Send` type as its parameter, and so it errors when we call it with something non-`Send`. We can do
this by abusing a tiny amount of "dead code" (code that will never be executed), so you can even
leave the following snippet in your code base.

```rust
// I could have called this `assert_send` but what's the fun in that?
fn help_me_figure_this_out(_: impl Send) {}

#[allow(dead_code)]
fn assert_my_handler_is_ok() {
    // This is the fun part: we need to call our handler, and *not* await it's Future.
    help_me_figure_this_out(page())
}
```

If we insert this into our example program, we get the following diagnostic (in addition to the
unhelpful one, but oh well, the world isn't perfect):

```
❯ cargo c
    Checking axum-async v0.1.0 (C:\_Hobby\axum-async)
error: future cannot be sent between threads safely
  --> src\main.rs:16:29
   |
16 |     help_me_figure_this_out(page())
   |                             ^^^^^^ future returned by `page` is not `Send`
   |
   = help: within `impl Future<Output = ()>`, the trait `Send` is not implemented for `Rc<()>`
note: future is not `Send` as this value is used across an await
  --> src\main.rs:9:34
   |
7  |     let rc = std::rc::Rc::new(());
   |         -- has type `Rc<()>` which is not `Send`
8  | 
9  |     sleep(Duration::from_secs(1)).await;
   |                                  ^^^^^^ await occurs here, with `rc` maybe used later
10 | }
   | - `rc` is later dropped here
note: required by a bound in `help_me_figure_this_out`
  --> src\main.rs:12:36
   |
12 | fn help_me_figure_this_out(_: impl Send) {}
   |                                    ^^^^ required by this bound in `help_me_figure_this_out`

[...]
```

And what if my handler takes any parameters?
--------------------------------------------

A valid question - real world code rarely looks like `page`. Fortunately, we don't *have* to do
anything - if `page` has parameters, the compiler will still emit the "hey, this isn't `Send`"
error. It will just be buried a bit more:

```rust
async fn page(req: Request<()>) {
    let rc = std::rc::Rc::new(());

    sleep(Duration::from_secs(1)).await;
}
```

```
❯ cargo c
    Checking axum-async v0.1.0 (C:\_Hobby\axum-async)
error[E0061]: this function takes 1 argument but 0 arguments were supplied
  --> src\main.rs:16:29
   |
16 |     help_me_figure_this_out(page())
   |                             ^^^^-- supplied 0 arguments
   |                             |
   |                             expected 1 argument
   |
note: function defined here
  --> src\main.rs:6:10
   |
6  | async fn page(req: Request<()>) {
   |          ^^^^ ----------------

error: future cannot be sent between threads safely
[...]
```

This is fine if we don't plan to keep the check functions in our code base. We can fix our original
issue, delete the code and continue with our lives. However, if we actually want our code to
compile, we can add the required number of parameters as `todo!()`s to our function, and it will
~~compile~~ error just fine. And so, our final solution:

```rust
async fn page(req: Request<()>) {
    let rc = std::rc::Rc::new(());

    sleep(Duration::from_secs(1)).await;
}

fn assert_send(_: impl Send) {}

#[allow(unreachable_code)] // prevent warnings because todo!()s make our dead code unreachable. Duh.
fn assert_my_handlers_are_ok() {
    // List all handlers here
    assert_send(page(todo!()));
}
```

[axum]: https://crates.io/crates/axum

[Send approximation]: https://rust-lang.github.io/async-book/07_workarounds/03_send_approximation.html
[async book]: https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html
