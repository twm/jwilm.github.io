---
title: Announcing Alacritty, a GPU-accelerated terminal emulator
date: 2017-01-05
tags: Rust, Alacritty
description: Initial source-only release of Alacritty
published: false
---

[Alacritty] is a blazing fast, GPU accelerated terminal emulator. It's written in
Rust and uses OpenGL for rendering to be the fastest terminal emulator
available. [Alacritty] is available on GitHub in source form. 

<img width="743" alt="Alacritty Screenshot" src="https://cloud.githubusercontent.com/assets/4285147/21690874/19037262-d32b-11e6-9c18-706b1f979eb7.png">

READMORE

<p align="center">
  <em>Alacritty running vim inside tmux</em>
</p>

# Alacritty

The rest of this post discusses what Alacritty is, why it was built, who it's
targeted at, and some architectural decisions that have enabled its unparalleled
performance. I'll be giving a technical talk at the January 2017 [Rust Meetup in
SF] if you want to learn more.

## About the project

Alacritty is the result of frustration with existing terminal emulators.
Using `vim` inside `tmux` in many terminals was a particularly bad experience.
Nothing was ever quite fast enough. Many Linux alternatives are actually decent.
For example, `urxvt` and `st` give good experiences. But they are difficult to
configure. The options for macOS are even worse, especially with a full-screen
terminal on a 4k monitor. None of these terminals are cross-platform--they are
usually married to the windowing and font rendering APIs of their native
platform.

Alacritty aims to address those issues. The project's architecture and features
are guided by a set of values:

1. **Correctness:** Alacritty should be able to properly render modern terminal
   applications like `tmux` and `vim`. Glyphs should be rendered properly, and
   the proper glyphs should be displayed.
2. **Performance:** Alacritty should be the fastest terminal emulator available
   anywhere.
3. **Appearance:** Alacritty should have beautiful font rendering and look
   fantastic on all supported platforms.
4. **Simplicity:** Alacritty should be conservative about which features it
   adds. As we've learned from past terminal emulators, it's far too easy to
   become bloated. `st` taught us that it doesn't need to be that way. Features
   like GUI-based configuration, tabs and scrollback are unnecessary. The latter
   features are better provided by a terminal multiplexer like `tmux`.
5. **Portability:** Alacritty should support major operating systems including
   Linux, macOS, and Windows.

## Initial Features

Many programs work correctly and without issue. `vim`, `tmux`, `htop`, various
pagers and many other full-screen applications are rendered properly.

Alacritty is incredibly fast. Compare `find /usr` or equivalent with
your favorite terminal emulator. Keep in mind that command will be faster the
second time it's run due to OS caching. Alacritty's performance scales well with
screen size. Running at larger resolutions will tip the scale further in
Alacritty's favor.

Alacritty's font rendering is great. Native font rasterization libraries are
used on each platform, and sub-pixel anti-aliasing is supported on both macOS
and Linux.

macOS and Linux are supported in this pre-alpha release. Windows is not yet part
of the list, but the initial offering demonstrates making a cross-platform
terminal emulator is possible.

Being a pre-alpha release, there is still pending work in key areas

* Less common applications have rendering issues
* A small subset of systems have performance issues with the OpenGL renderer
* Font rendering on macOS is not as good as the competition
* Wayland is not natively supported
* Fallback fonts are not supported
* Full-screen mode is not supported on Linux

Such issues will be resolved prior to a 1.0 release. Many of them will be
resolved long before then.

# What makes Alacritty fast

Alacritty's biggest claim is that it's the fastest terminal emulator available.
If there's a case where it's not, then it's either a bug in Alacritty or a
misconfigured system. Alacritty is fast for two reasons--the OpenGL renderer
and the high throughput parser.

## OpenGL Rendering

Alacritty's renderer is capable of doing ~500 FPS with a large screen
full of text. This is made possible by efficient OpenGL usage. State changes are
minimized as much as possible. Glyphs are rasterized only once and stored in a
texture atlas. Instance data for glyphs is uploaded once per frame, and the
screen is rendered in only two draw calls.

Alacritty isn't concerned with trying to only redraw what's necessary. The
entire screen is redrawn each frame because it's so cheap.

Nominally, Alacritty will draw a new frame whenever the terminal state changes,
and only when the state changes. Alacritty will simply sit idle when there's no
new data from the pseudoterminal or input events from the users. This helps
significantly with battery life.

Alacritty is very good at processing huge amounts of text. Say that a user wants
to `cat 1gb_file.txt`. There's a lot of work for the parser to do. If Alacritty
were drawing frames constantly on every change, it would take a long time to
finish parsing and rendering the contents. Thankfully, V-Sync can help here.

V-Sync limits the frames drawn by Alacritty up to the monitor's refresh
rate. 60Hz is a typical value here. Using this number, there are 16.7ms
available per frame. 2ms of that budget is occupied by the renderer. The
remaining 14.7ms are available to the parser to process that huge stream of
text. Any text that is added to the terminal and scrolled past between frames
will never be drawn. This is a **huge** performance win! The parser can process
many MBs of data between frames, and the user will still see text drawn very
smoothly.

## The Parser

Alacritty's performance is enhanced by having a good parser. Rust helped
significantly with this by enabling us to write small testable components and
combine them without overhead.

### Zero-cost abstractions

Rust's zero-cost abstractions make it possible to build nicely abstracted
components and then combine them together as if they were one big hand-written
state machine.

The parser has an `advance()` method which advances the state and sometimes
dispatches actions to the `Perform`. Here's the signature:

```rust
pub fn advance<P: Perform>(&mut self, performer: &mut P, byte: u8);
```

Whenever a byte arrives from the pseudoterminal, it is passed to this advance
method. Some bytes will cause state to accumulate in the parser; other bytes
will trigger an action. Actions might be something like printing a character to
the screen or executing a CSI escape sequence. These actions are defined on a
`Perform` type which is passed to `advance()`. For example, here's the first
couple of methods on the trait:

```rust
/// Performs actions requested by the Parser
pub trait Perform {
    /// Draw a character to the screen
    fn print(&mut self, char);

    /// Execute a C0 or C1 control function
    fn execute(&mut self, byte: u8);

    // ..
}
```

To see how this turns into a zero-cost abstraction, let's look at where the
`Perform` methods are actually called:

```rust
match action {
    Action::Print => performer.print(byte as char),
    Action::Execute => performer.execute(byte),
    // ..
}
```

Assuming the concrete `Perform` type requests `#[inline]` on these methods, they
will likely be inlined at the call site here and avoid function call overhead.

This same pattern is used for multiple layers of abstraction. Alacritty's
`vte::Perform` impl actually delegates to an `ansi::Handler` type for actions
which requires additional parsing (eg. `csi_dispatch()`).  The result is that we
get nice abstractions for things like `vte::Perform` and `ansi::Handler`, and
the parsing code compiles into what looks like a big hand-written state machine.

### Table-driven parsing

Both the [utf8parse] and [vte] crates that were written for Alacritty use
table-driven parsers. The cool thing about these is that they have *very* little
branching; [utf8parse] only has [one branch][utf8parse branch] in the entire
library!

The transition tables for both `vte` and `utf8parse` are written using a
procedural macro. The macro translates a [high-level definition] of the state
machine transitions to [low-level lookup tables]. That is, the state machine can
be described like this:

```rust
// Transition tables description. Much was omitted for brevity.
pub static STATE_CHANGE: [[u8; 256]; 16] = vt_state_table! {
    State::Anywhere => {
        0x80...0x8f => (Action::Execute, State::Ground),
        0x9b        => State::CsiEntry,
    },
    State::Ground => {
        0x00...0x17 => Action::Execute,
        0x20...0x7f => Action::Print,
        0xf0...0xf4 => (State::Utf8, Action::BeginUtf8),
    },
    State::Escape => {
        0x00...0x17 => Action::Execute,
        0x30...0x4f => (Action::EscDispatch, State::Ground),
        0x58        => State::SosPmApcString,
    },
}
```

The output of the procedural macro is the transition tables:

```rust
// Transition tables
pub static STATE_CHANGE: [[u8; 256]; 16] = [
    [
        0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8,
        0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8, 0u8,
        0u8, 0u8, 0u8, 0u8, 92u8, 0u8, 92u8, 10u8, 0u8, 0u8,
        // ..
    ],
    // ..
];
```

Whenever a byte arrives, the parser looks up the table for the _current_ state
and indexes with that byte. The value returned contains the _next_ state and any
action required to make that transition. This process doesn't requires
branching--it's a simple, fast lookup.

# New libraries

Developing Alacritty required several pieces of library infrastructure which was
not available. A non-GPL licensed cross-platform clipboard library, a `vte`
parser, cross-platform font rasterization and `fontconfig` bindings were all
needed to build this project. [vte] and [utf8parse] have been published on
[crates.io]. The remaining libraries are still in Alacritty's source tree and
will be published independently at some point.

Here's a quick run-down of the new libraries:

- [copypasta]: Cross-platform clipboard access library
- [font]: Cross-platform font rasterization library
- [utf8parse]: A table-driven UTF-8 parser
- [vte]: A table-driven terminal protocol parser
- [ffi-util]: Utilities for simplifying wrapping FFI types. This project is
  from [@sfackler]'s [rust-openssl] that I've put into a module
  for wrapping `fontconfig` C types.

# Conclusion

Alacritty is a very fast and usable terminal emulator. Although this is a
pre-alpha release, it works well enough for many developers to be used as a
daily driver. I've personally been using it as my primary terminal for a few
months now.

For those wanting to learn more about the project, I'll be giving a talk about
Alacritty at the upcoming [Rust Meetup in SF] on January 19. I'm also planning
some more technical blog posts about various subsystems of Alacritty.

[Alacritty]'s source is available on GitHub. Try it out for yourself!

# Graveyard

Once Alacritty reaches an alpha level of readiness, precompiled binaries will be
provided for supported operating systems.



[Alacritty]: https://github.com/jwilm/alacritty
[copypasta]: https://github.com/jwilm/alacritty/tree/master/copypasta
[font]: https://github.com/jwilm/alacritty/tree/master/font
[utf8parse]: https://docs.rs/utf8parse
[vte]: https://docs.rs/vte
[ffi-util]: https://github.com/jwilm/alacritty/tree/master/ffi-util
[utf8parse branch]: https://github.com/jwilm/vte/blob/b016827f471041320996f3273b4e3058501d7edf/utf8parse/src/lib.rs#L61
[state transition definitions]: https://github.com/jwilm/vte/blob/b016827f471041320996f3273b4e3058501d7edf/src/table.rs.in
[Rust Meetup in SF]: https://www.meetup.com/Rust-Bay-Area/events/236668916/
[`vte::Parser`]: https://docs.rs/vte/
[crates.io]: https://crates.io
[rust-openssl]: https://github.com/sfackler/rust-openssl
[@sfackler]: https://github.com/sfackler
[high-level definition]: https://github.com/jwilm/vte/blob/master/src/table.rs.in
[low-level lookup tables]: https://github.com/jwilm/vte/blob/master/src/table.rs

<!--
Other stuff that would be fun to talk about:

- The grid container and its index types
- Texture atlases
- Config loading
- Ref tests
- VTE state table definitions

-->
