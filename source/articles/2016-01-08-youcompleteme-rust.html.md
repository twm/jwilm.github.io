---
title: YouCompleteMe, Rust
date: 2016-01-08
tags: Vim, YouCompleteMe, Rust
slug: /youcompleteme-rust
---

[YouCompleteMe][] now supports Rust auto-completion and GoTo. Rust semantic
analysis is provided by [racerd][], a JSON/HTTP server powered by [racer][]. YCM
with Rust provides a superior experience to current vim-racer, emacs-racer, and
other racer-based plugins.

READMORE

[![ycm_rust_launch.gif](https://d23f6h5jpj26xu.cloudfront.net/xtrifoeons6nfa_small.gif)](http://img.svbtle.com/xtrifoeons6nfa.gif)

# YouCompleteMe

For the uninitiated, [YouCompleteMe][] is a fast, fuzzy, as-you-type
code-completion engine built originally for Vim. YCM runs on Mac, Linux, and
Windows and includes completion engines for C/C++, ObjC, Python, C#, Go,
TypeScript, JavaScript, and now Rust. YCM additionally provides an identifier
based completion engine to supplement semantic completers and provide
completions for languages without native support.

The YCM core, [ycmd][], exists [as an independent project][ycmd-announcement];
clients for [Vim][YouCompleteMe], [Emacs][emacs client], and [Atom][atom client]
all share the same infrastructure. The complete list of known clients is found
in [the ycmd README][known clients]. If a ycmd client does not exist for
`$EDITOR`, integrating with ycmd is as simple as integrating with a REST API.
The [ycmd example client][] shows how it's done.

# Rust

> [Rust](http://rust-lang.org) is a systems programming language that runs
> blazingly fast, prevents segfaults, and guarantees thread safety. 

Support for Rust is now available in YCM. The result is a powerful development
environment providing completions and GoTo in your favorite editor. Examples
below assume your editor is Vim. For other editors, please reference their YCM
client.

## Completions

Semantic completions in YCM are provided when a *semantic trigger*, `.`, or
`::`, is detected. In the provided example, you can see a completion menu
immediately appear after typing `std::`, `std::collections::`, and `Vec::`; no
hotkey was necessary. As you continue to type, YCM fuzzy-filters available
completions using your input to narrow the completion list.

## GoTo

The YCM Rust Completer provides the `GoTo` subcommand. `GoTo` attempts to find
where the identifier under the cursor is defined. If successful, a buffer is
opened for the file containing the definition, and the cursor is placed on the
definition.

The complete GoTo command in Vim is `:YcmCompleter GoTo`, and I highly recommend
mapping it to some hotkey. Example:

```viml
nnoremap <Leader>] :YcmCompleter GoTo<CR>
```

## YCM Configuration (Vim)

Rust completions and GoTo from the current crate and its dependencies will be
available without any additional configuration. For completions in the standard
library, a single variable `g:ycm_rust_src_path` must be defined in your .vimrc.

```viml
" Naturally, this needs to be set to wherever your rust
" source tree resides.
let g:ycm_rust_src_path = '/usr/local/rust/rustc-1.5.0/src'
```

# Differences from vim-racer, emacs-racer, and others

Since `racerd`, and subsequently YCM's Rust Completer are powered by `racer`,
the same code-completion and find-definition features are available. The
addition of YCM to this equation provides a *massive* quality-of-life
improvement over the existing plugins:

- No hotkey required for completions
- Fuzzy search of available completions
- Identifier-based completion engine to supplement rust completions
- Performance: Completions are cached within YCM, and latency with racerd is
  typically only a few ms once files are cached from disk. This is discussed
  more in the racerd section following.

# racerd

[racerd][] is suitable for providing completions and find-definition support for
*any* Rust IDE and is not constrained to YCM. It provides several benefits over
integrating racer via the command line or with the racer `daemon` flag.

1. Persistent file caching: When starting a new racer process for every
   completion, or when running racer in daemon mode, the cache is thrown out
   after each operation. Racerd keeps this cache between requests. This gives a
   nice performance boost since files only need to be read from disk once.

2. Support for dirty buffers: [Racerd's JSON API][] supports a `buffers` field -
   a list of file paths (which need not exist) and associated contents which is
   added to the racer cache before performing a query. This feature eliminates
   the need for temporary files.

3. HTTP/JSON API: Very effective method for integrating a semantic completion
   engine. Just about every language has a built-in or library for querying such
   an API, and it is extremely performant on localhost (a few ms for completing
   out of standard library). The process is long lived; there is no process
   startup overhead for each query. Racerd also has [extensive API
   documentation][Racerd's JSON API].

# Giving Thanks

YouCompleteMe would not be the fantastic project it is, both technically and as
a positive open-source community, without the work of [@Valloric][] and the YCM
team [@micbou][], [@oblitum][], [@vheon][], and [@puremourning]. [racerd][] and
YCM's Rust completer would not have been possible without [@phildawes][]'
fantastic [racer][] library. Thanks to [@birkenfeld][] for his assistance
implementing some changes in racer to make racerd possible. Thanks to [@reem][]
whom I pestered with far too many questions in #*iron*. Finally, thanks to the
YCM team for reading drafts of this post.


[other editors]: https://github.com/Valloric/ycmd#known-ycmd-clients
[ycmd-announcement]: https://val.markovic.io/articles/youcompleteme-as-a-server
[ycmd]: https://github.com/valloric/ycmd
[racer]: https://github.com/phildawes/racer
[racerd]: https://github.com/jwilm/racerd
[YouCompleteMe]: https://github.com/valloric/YouCompleteMe
[emacs client]: https://github.com/abingham/emacs-ycmd
[atom client]: https://atom.io/packages/you-complete-me
[ycmd example client]: https://github.com/Valloric/ycmd/tree/master/examples
[Racerd's JSON API]: https://github.com/jwilm/racerd/blob/master/docs/API.md
[known clients]: https://github.com/Valloric/ycmd#known-ycmd-clients

[@micbou]: https://github.com/micbou
[@Valloric]: https://github.com/valloric
[@phildawes]: https://github.com/phildawes
[@oblitum]: https://github.com/oblitum
[@puremourning]: https://github.com/puremourning
[@vheon]: https://github.com/vheon 
[@birkenfeld]: https://github.com/birkenfeld
[@reem]: https://github.com/reem 
