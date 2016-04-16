---
title: Rust Snippets
date: 2016-03-27
tags: Vim, Rust
published: false
slug: /rust-snippets
---

Rust saves you from a lot of really terrible things, but one thing is
unescapable - boilerplate. How much code do you really need to write when you
`impl From` for something? Most of that is interface, and there's 1 line of
meat. Even worse, implementing a new Error type requires implementing
`error::Error` which has two methods, and `::fmt::Display` which has another
method. Having [just shipped OnePush][OnePush Announcement], I've had a bit of
extra time to remove some friction from writing Rust code.

If you feel this same pain, read on. Here I present a quick-start guide for Rust
snippets in Vim - batteries included.

READMORE

Enter [UltiSnips][]

<div id="asciicast">
  <script type="text/javascript" src="https://asciinema.org/a/316t2p7xbd0ym6rsg03x6lue2.js" id="asciicast-316t2p7xbd0ym6rsg03x6lue2" async></script>
</div>

Hello, rust snippets.

[OnePush Announcement]: (https://onesignal.com/blog/announcing-our-new-delivery-backend/)
[UltiSnips]: (https://onesignal.com/blog/announcing-our-new-delivery-backend/)
