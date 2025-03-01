---
layout: post
title:  "Remote server functionality in probe-rs"
date:   2025-02-20 21:26 +0200
categories: rust probe-rs
excerpt_separator: <!--more-->
---

We have recently merged a series of exciting patches[^1] [^2] [^3] [^4] into [probe-rs], that
implement server-client functionality using websockets and postcard-rpc. In this post, I'll try
to explain the motivation behind the work, the design, configuration and usage of this feature.

## Why?

I've been recently given the opportunity to try [teleprobe]. Teleprobe is the software that powers
the [embassy] HIL test rig, and it gives a very convenient way to run a cargo project on an MCU that
is connected to a remote machine. I was pretty impressed with the experience it gave me, and since
teleprobe builds on top of probe-rs, I thought why shouldn't we integrate a similar functionality
into probe-rs itself?

<!--more-->

Besides the cool factor, remote access means I don't have to carry my boards with me, I can share
my hardware park with others, and my PCs are isolated from the boards. I can also just grab my
laptop and move around to a more comfortable place in my apartment to do work, without fooling
around with cables. In theory, I can also use it to connect to boards from inside WSL, without the
overhead of USB/IP.

Did I mention the cool factor?

## How?

The change affects the probe-rs CLI (probe-rs-tools). I tried my best to not introduce changes into
the library itself, although I did end up redesigning how progress reporting works during device
flashing.

From very early on, websockets as a transport was a natural choice. I wanted the server to implement
a web server that served a status page, and if we already had a server in place, why would we use
another transport for the communication itself?

In broad strokes, the architecture I wanted looks like this:

```
                                                                    ┌────────────────┐
    Local operation                                              ┌─►│Function handler│
                                                                 │  └────────────────┘
   ┌────────┐     ┌────────────────┐            ┌──────────────┐ │
   │probe-rs│     │     Client     │   Channel  │    Server    │ │  ┌────────────────┐
   │  CLI   │◄───►│ implementation │◄──────────►│ and dispatch │◄┼─►│Function handler│
   └────────┘     └────────────────┘            └──────────────┘ │  └────────────────┘
                                                                 │
                                                                 │  ┌────────────────┐
                                                                 └─►│Function handler│
                                                                    └────────────────┘

   ──────────────────────────────────────────────────────────────────────────────────

                                                                    ┌────────────────┐
    Remote operation                                             ┌─►│Function handler│
                                                                 │  └────────────────┘
   ┌────────┐     ┌────────────────┐            ┌──────────────┐ │
   │probe-rs│     │     Client     │  Websocket │    Server    │ │  ┌────────────────┐
   │  CLI   │◄───►│ implementation │◄──────────►│ and dispatch │◄┼─►│Function handler│
   └────────┘     └────────────────┘            └──────────────┘ │  └────────────────┘
                                                                 │
                                                                 │  ┌────────────────┐
                                                                 └─►│Function handler│
                                                                    └────────────────┘
```

The idea has always been to keep duplication to a minimum. There is a single CLI codebase, there is
a single server/RPC codebase, only the transport is different between local and remote sessions.
This architecture has a nice benefit: we can write multiple frontends for the same server, which
means we can easily make `cargo embed` remote-capable, too, or someone can write a probe-rs GUI
application, or more!

It took me three attempts to reach the current state of the implementation. The first attempt was
a simple "let's pipe stdin/stdout over websockets" implementation, which was simple enough, but
very limited and not the greatest UX. For example, it wasn't able to display progress bars. There
was some push back against the idea, so after a few days of playing with it, I moved on to the
second attempt.

For the second attempt, I've rolled my own, rather simplistic RPC implementation. At first, I used
JSON to serialize data, my own simple dispatch method, and absolutely no compatibility checks
between the server and the client. While it worked okay, I quickly ran into a few problems:

- I did not implement any sort of versioning into the API, so different probe-rs versions could
  silently miscommunicate. Not great.
- I had some very smelly hacks in the code, to get command cancellation working. The implementation
  worked, but I expect nobody would have merged a PR that blocked tokio in a `Drop` implementation.

In general, this attempt showed me that an RPC implementation was viable, but I needed something
more structured than my own terrible non-solution.

For the third attempt, I chose [postcard-rpc]. Postcard is a simple, embedded-focused serialization
format that I've been meaning to try for a long time, and this was the perfect opportunity. Postcard
RPC is, as the name suggests, built on top of Postcard and Postcard Schema, to provide the bits I
needed: a client, dispatch, the possibility to implement my own transport, and best of all, a very
friendly author!

After the second attempt, reworking the code on top of Postcard RPC actually took a bit of time.
While most of the changes were mechanical at this point (not counting actually trying to come up
with an okay-looking commit history), I had a few challenges:
- I needed to find a way to implement a transport layer that works both remotely and locally. I
ended up implementing `tokio` `Channel` objects as `WireTx` and `WireRx`, and a bit of code around
them to either connect the channels in process memory, or pipe messages through websockets.
- I needed a way to make sure messages aren't lost when the server streamed lots of messages in a
topic. PRPC had a concept that allowed me to implement this, I merely had to [undeprecate] exclusive
subscribers and implement a global timeout to throttle messages with.
- Lastly, I ran into a bug that only manifested on low performance machines, and resulted in my client
freezing completely. Thanks James for [fixing it] quickly!

Besides Postcard RPC, probe-rs itself posed a few challenges for me:
- Some tasks, for example flashing firmware, take a long time. We receive progress updates through
callbacks, but then we have to somehow communicate them to the client and display them on a progress
bar, without locking everything up. I wrote a utility function that uses
`tokio::task::spawn_blocking` to run the blocking code, and used channels to stream events out of
the task.
- These long-running tasks need to be cancellable. The user might press ctrl-c at any time, and we
shouldn't keep the server busy running something when the client has gone away. I added a `Cancel`
topic that the client can write, and the server manages a cancellation token that can gracefully, or
sometimes more forcefully end a function. I probably didn't get this right just yet, as the
cancellation can take some time to actually become effective, and the scheme isn't implemented for
everything anyway, but future improvements are okay.

## What's left to do here?

The work is far from finished. Not every function has been ported: commands I don't personally use
like `profile`, `benchmark`, `trace`, as well as more complex ones like `gdb`, `debug` and
`dap-server` need to be rewritten. `cargo embed` and `cargo flash` also can be updated to use the
RPC client. Currently they are included in the `probe-rs-tools` binary to reduce code duplication,
but if we extract the RPC implementation, they can be split out again without losing any
functionality.

All right, long story, but we've arrived at the interesting part: how can we use this?

## Installation

For now, the server/client functions are not enabled by default. This means you'll have to install
probe-rs [from source], with the `remote` feature enabled. If you have all the prerequisites, and
you are compiling for your machine (as opposed to, for example, cross-compiling for a Raspberry Pi),
the command you would use is:

```bash
cargo install probe-rs-tools --git https://github.com/probe-rs/probe-rs --locked --features remote
```

You will need to do this installation for both your client machine, and your server. If your server
is a Raspberry Pi or similar device that is not able to compile probe-rs itself, please take a look
at the [cross-compilation] guide!

## Configuration

Before starting the server, you will need to configure it. To do so, create a `.probe-rs.toml`
(or YAML, or JSON, but I'll be using TOML in this article) file in your home folder on the server.
Currently the only thing you can define is a list of users, but in the future we'll have more
options. For each user you will need to assign a unique token. Clients will need to specify this
token to connect. Keep these tokens secret, and make sure the users do the same.

Your configuration may look like this:

```toml
[[server_users]]
name = "danielb"
token = "asbd"

[[server_users]]
name = "johnc"
token = "alskdhsaoighew"
```

On the client side, the configuration file is optional, but it can be used to make working with
remotes and multiple devices easier. Client configuration is done using `preset`s, which are named
bundles of parameters. These parameters act exactly as arguments to the command-line program. For
example, the following configuration file defines two presets, both of them configure probe-rs to
work with different devices:

```toml
[presets]
esp32c2 = { probe = "0403:6010:FT929K8X" }
stm32f051r8-jlink = { probe = "1366:0101:000778807372", speed = 1000, chip = "stm32f051r8tx" }
```

You can select etween these presets by passing `--preset <NAME>` to the CLI, like
`probe-rs run --preset esp32c2`. This will then be equivalent to you running
`probe-rs run --probe "0403:6010:FT929K8X"`.

## Running commands remotely

On the server side, you don't need a lot to do. Connect your probes, make sure the machine is
visible on your network, then run `probe-rs serve`. This will load the configuration, then start
the server.

You will then be able to open `http://<servers-hostname>:3000` in your browser for a simple page
that lists the connected probes. You can use this page to verify that the server is running and is
accessible as expected.

For remote access, services like `ngrok` work well enough, although you should be aware that the
client does not do any TLS certificate verification currently.

Once you have a server configured and up and running, you'll be able to use the server's hostname
and one of the user tokens to connect to it. You can specify the remote host with `--host`, and the
token with `--token`. The host name will need the `ws://` (websocket) or `wss://` (websocket over
TLS) schema, and port `3000`.

For example, if your server's host name is `probe-rs-server`, you would write the following:

`probe-rs run <path-to-your-elf> --chip stm32f051r8tx --host ws://probe-rs-server:3000 --token asbd`

Alternatively, you can add the host and token to the configuration presets, and then you can just
run:

`probe-rs run <path-to-your-elf> --preset stm32f051r8tx-jlink`

## Limitations

`--chip-description-path` is not supported remotely. We plan on enabling this later, but it needs
some amount of probe-rs library work.

Not every command is supported remotely. At the time of writing this post, the list of commands that
are working remotely is:

- `list`
- `read`
- `write`
- `reset`
- `chip`
- `info`
- `download`
- `attach`
- `run`
- `erase`
- `verify`

You will also not be able to see logs generated by the server, or generate error reports.

I hope this will turn out to be as useful for you as it is for me. If you have questions or
suggestions, drop by in the [#probe-rs] matrix server, and let's chat!

[probe-rs]: https://github.com/probe-rs/probe-rs
[teleprobe]: https://github.com/embassy-rs/teleprobe/
[embassy]: https://github.com/embassy-rs/embassy
[postcard-rpc]: https://github.com/jamesmunns/postcard-rpc
[undeprecate]: https://github.com/jamesmunns/postcard-rpc/pull/82
[fixing it]: https://github.com/jamesmunns/postcard-rpc/pull/88
[from source]: https://probe.rs/docs/getting-started/installation/#installing-from-source-(cargo-install)
[cross-compilation]: https://probe.rs/docs/library/crosscompiling/
[#probe-rs]: https://matrix.to/#/#probe-rs:matrix.org
[^1]: https://github.com/probe-rs/probe-rs/pull/3003
[^2]: https://github.com/probe-rs/probe-rs/pull/3049
[^3]: https://github.com/probe-rs/probe-rs/pull/3079
[^4]: https://github.com/probe-rs/probe-rs/pull/3081
