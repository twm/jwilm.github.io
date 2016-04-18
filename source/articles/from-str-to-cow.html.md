---
title: From &str to Cow
date: 2016-04-16
tags: Rust
---

Some of the first Rust code I wrote was a struct with a `&str` field. As you
might imagine, the borrow checker didn't let me do a lot of things, and the API
ergonomics were limited. This article aims to demonstrate the issues with having
raw `&str` references in structs, introduce some intermediate APIs that
alleviate the ergonomics but aren't necessarily efficient, and end with an
implementation that is both ergonomic and efficient.

READMORE

Let's say we're making a library for the _example.com_ API. Each API call must
be authenticated with a token. A token definition might look like this:

```rust
/// Token for example.io API
pub struct Token<'a> {
    raw: &'a str,
}
```

And maybe there's an impl for creating a token given a `&str`.

```rust
impl<'a> Token<'a> {
    pub fn new(raw: &'a str) -> Token<'a> {
        Token { raw: raw }
    }
}
```

This `Token` impl works really well for `&'static str`. However, what if a user
doesn't want to embed their authentication secret in the binary? Maybe the
secret is loaded from a vault at run time. We might want to write some code like
this.

```rust
/// Imagine that this function exists
let secret: String = secret_from_vault("api.example.io");
let token = Token::new(&secret[..]);
```

This implementation has a big limitation; `token` cannot outlive `secret`, and
that means `token` can't escape this stack frame. What if `Token` just held a
`String`? That would get rid of the lifetime parameter making `Token` a purely
owned type.

`Token` and its `new` function look like this after making the change.

```rust
struct Token {
    raw: String,
}

impl Token {
    pub fn new(raw: String) -> Token {
        Token { raw: raw }
    }
}
```

The use case where a String is provided has been fixed.

```rust
// this works now
let token = Token::new(secret_from_vault("api.example.io"))
```

However, this has actively harmed the usability of providing a `&'str`. For
example, this won't compile:

```rust
// doesn't compile
let token = Token::new("abc123");
```

The consumer of this API would need to manually convert that into a `String`
first.

```rust
let token = Token::new(String::from("abc123"));
```

If `new` took a `&str` instead of a `String`, it could hide `String::from` in
the implementation. However, passing in a `String` would become less ergonomic,
and it would involve and additional heap allocation. Let's see what that
looks like:

```rust
// new function now looks like so
impl Token {
    pub fn new(raw: &str) -> Token {
        Token(String::from(raw))
    }
}

// &str can be passed seamlessly
let token = Token::new("abc123");

// Can still use a String, but it needs to be sliced, and
// the new fn will copy it
let secret = secret_from_vault("api.example.io");
let token = Token::new(&secret[..]); // inefficent!
```

There is a way that `new` can accept both forms with no allocation needed in the
case of a String.

# Introducing Into

The standard libary has a trait
[Into](https://doc.rust-lang.org/std/convert/trait.Into.html) which will help
our `new` problem. The trait definition looks like this:

```rust
pub trait Into<T> {
    fn into(self) -> T;
}
```

The `into` function it defines is pretty straight-forward; it consumes self (the
thing implementing Into) and returns a `T` (note the type parameter on the trait
definition). Here's how it's used:

```rust
impl Token {
    /// Create a new token
    ///
    /// Can be passed either a &str or String
    pub fn new<S>(raw: S) -> Token
        where S: Into<String>
    {
        Token { raw: raw.into() }
    }
}

// &str
let token = Token::new("abc123");

// String
let token = Token::new(secret_from_vault("api.example.io"));
```

There's a lot going on here. First, the function now has a generic type
parameter, `S`. The argument `raw` has this type. The line reading `where S:
Into<String>` limits the types of `S` to *anything* that implements
`Into<String>`. Since the standard library already provides `Into<String>` for
`&str` and for `String`, our use case is handled. ^[1](#1)

Although the ergonomics have been greatly improved, there's still an issue with
this API. Passing a `&str` to `new` requires an allocation to store the value as
a `String`.

# Cow to the rescue

The standard library has a type
[`std::borrow:Cow`](https://doc.rust-lang.org/std/borrow/enum.Cow.html) which
enables us to keep the ergonomics of the `Into<String>` API while also allowing
for borrowed values like a `&str`.

Here's the scary-looking definition of `Cow`:

```rust
pub enum Cow<'a, B> where B: 'a + ToOwned + ?Sized {
    Borrowed(&'a B),
    Owned(B::Owned),
}
```

Let's break that down.

- `Cow<'a, B>` has two generic parameters; a lifetime `'a`, and some type `B`
- `B` is constrained to `'a + ToOwned + ?Sized`
    - `'a` - `B` cannot contain a lifetime that outlives `'a`
    - `+ ToOwned` - `B` must implement `ToOwned`.
    - `+ ?Sized` - The size of B can be unknown at compile time. This isn't
        relevant for our use case, but it means that trait objects may be used
        with the Cow type.
- There's two variants
    - `Borrowed(&'a B)` - a reference to some object of type B. The lifetime of
        this reference is the same as the lifetime bound.
    - `Owned(B::Owned)` - The `ToOwned` trait has an associated type `Owned`.
        This variant holds that type.

We want to have a `Cow<'a, str>`, which will look something like this after type
substitution.

```rust
enum Cow<'a, str> {
    Borrowed(&'a str),
    Owned(String),
}
```

In short, `Cow<'a, str>` will be either a `&str` with the lifetime `'a`, or it
will be a `String` which is not bound by that lifetime. This sounds great for
the Token type! It will be able to hold either a `&str` or a `String`.

```rust
struct Token<'a> {
    raw: Cow<'a, str>
}

impl<'a> Token<'a> {
    pub fn new(raw: Cow<'a, str>) -> Token<'a> {
        Token { raw: raw }
    }
}

// Create the tokens.
let token = Token::new(Cow::Borrowed("abc123"));
let secret: String = secret_from_vault("api.example.io");
let token = Token::new(Cow::Owned(secret));
```

A Token can now be created with either an owned or a borrowed type, but we've
lost the API ergonomics! `Into` can do the same thing for our `Cow<'a, str>` as
it did for a simple `String` earlier. The final Token implementation looks like
this:

```rust
struct Token<'a> {
    raw: Cow<'a, str>
}

impl<'a> Token<'a> {
    pub fn new<S>(raw: S) -> Token<'a>
        where S: Into<Cow<'a, str>>
    {
        Token { raw: raw.into() }
    }
}

// Create the tokens.
let token = Token::new("abc123");
let token = Token::new(secret_from_vault("api.example.io"));
```

Now, a token can be created ergonomically from either a `&str` or a `String`.
The lifetime bound on `Token` is no longer a problem for escaping stack frames
when created with a `String` or `&'static str`; it can even be sent across
threads!

```rust
let raw = String::from("abc");
let token_owned = Token::new(raw);
let token_static = Token::new("123");

thread::spawn(move || {
    println!("token_owned: {:?}", token_owned);
    println!("token_static: {:?}", token_static);
}).join().unwrap();
```

Trying to send a token with a non-`'static` ref will fail.

```rust
// Make a ref with non-'static lifetime
let raw = String::from("abc");
let s = &raw[..];
let token = Token::new(s);

// This won't work
thread::spawn(move || {
    println!("token: {:?}", token);
}).join().unwrap();
```

Indeed, the above example fails with

```
error: `raw` does not live long enough
```

If you're hungry for more examples, please check out the [PagerDuty API
client](https://github.com/jwilm/pagerduty-rs) which uses `Cow` extensively.

Thanks for reading!

# Notes

## 1

If you were to go looking for the `Into<String>` impl for `&str` and `String`,
you wouldn't find it. This is because there's a generic implementation of Into
for types implementing From. It's often said that `From` implies `Into`, and
it's because of this blanket impl. The whole thing looks like this

```rust
impl<T, U> Into<U> for T where U: From<T> {
    fn into(self) -> U {
        U::from(self)
    }
}
```

