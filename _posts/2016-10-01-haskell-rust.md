---
layout: post
title: FFI with Haskell and Rust
---

I've had an idea percolating for a while. I love using Haskell for work.
It's functional, expressive, and easy to reason about. It's also
strongly typed and deals with immutable data (for the most part) which are
big pluses for writing good code. However, it's garbage collected,
and while it's not likely to run into performance issues for small
projects, on larger ones it can get in the way. Commonly with Haskell,
if one really needed speed, the FFI would be used with C code in order
to get the performance needed. The only problem with that is that C is
unsafe with undefined behavior being the norm. This is where Rust is
great alternative C for those performance critical functions needed in
some Haskell problems. If you've ever used Haskell before you know that
the documentation is either outdated, unrelated to what you actually
want, or just plain missing. I spent hours digging through old Haskell and
Rust blog posts, stack overflow posts from 2009, GHC options, and god
knows how many compiler errors. Rather than having you dig through the depths
of the archaeological nightmare that is the Internet, I've decided to
write this for anyone trying to use Rust in Haskell. Let me be clear,
this article is not covering how to use Haskell in Rust. That's for some
other time.

Let's get started then.

### Tooling
This articles assumes you have the following installed:

- rustc - at least 1.12 stable
- cargo
- cabal
- GHC
- make

Why not stack? It kept yelling at me for weird things. I like it but
this is a simple example and cabal will suffice. Besides stack is built
on top of cabal so you should already have it anyways.

### Setting up our Rust Code
First let's setup our project directory. Initialize it with the
following command:

```bash
cargo new haskellrs
```

This well setup a folder haskellrs that's laid out like this:

```
.
├── Cargo.toml
└── src
    └── lib.rs
```

Before we start setting up our Haskell code as well let's get our Rust
code straightened out. First we need to modify our Cargo.toml file. When
you open it up it should look something like this:

```toml
[package]
name = "haskellrs"
version = "0.1.0"
authors = ["Michael Gattozzi <mgattozzi@gmail.com>"]

[dependencies]
```

We need to modify it to look like this:

```toml
[package]
name = "haskellrs"
version = "0.1.0"
authors = ["Michael Gattozzi <mgattozzi@gmail.com>"]

[lib]
name = "rusty"
crate-type = ["staticlib"]

[dependencies]
```

If you've done FFI before you might have noticed I chose to statically
link this file rather than dynamically link it. I have yet to get an
example working with dylib or the new cdylib format for Rust and would love to
know how if you've done it before. The three new lines added let Rust
know a few things:

1) The output file will be a .a file called librusty.a (The common
convention for C code that's a library to start with lib and then the
name of the project).
2) This file can get statically linked into other programs because we've
defined it to be a static lib.
3) It knows how to setup the file differently with it's symbols for
other programs. Rust normally uses .rlib files to link Rust code to
other Rust code when you pull in external dependencies with Cargo. We
don't want that since Haskell would have no idea how to understand that
file format.

Okay so our project is setup! Let's start writing some Rust! Open up
your src/lib.rs file. It should look like this:

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
    }
}
```

We don't need this for our purposes so feel free to delete it. Instead
we're going to write two simple functions so that we can see how this
works. One with Strings and one with i32. First up we're going to write
a function called double_input that takes an i32 and doubles it. It
should look like this:

```rust
#[no_mangle]
pub extern "C" fn double_input(x: i32) -> i32 {
    2 * x
}
```

Let's go through each line:

1) The #[no_mangle] tells the Rust compiler not to do anything weird
with the symbols of this function when compiled because we need to be
able to call it from other languages. This is needed if you plan on
doing any FFI. Not doing so means you won't be able to reference it in
other languages
2) Now our actual function header. `pub` makes it available to use
elsewhere. `extern` means this is externally available outside our
library. `"C"` tells the compiler to follow the C calling convention when
compiling. If you don't know what that means, it has to do with how code
leaves values available for functions that called it on the CPU level.
You can find more information about it
[here](https://en.wikipedia.org/wiki/X86_calling_conventions#cdecl) if
you are interested on learning more. We need to do this so Haskell knows
how to treat the Rust code. For all intents and purposes Haskell thinks
it's calling C compiled code and not Rust. Now we finish with the rest
of the function. `fn` tells us this is a function we are declaring,
`double_input` is the function name `(x: i32)` means it has an input `x` of
type `i32` and `-> i32` means we are returning an `i32`.
3) Our next line is simple take `x` and multiply it by 2. Since it's the
last line in the Rust function by omitting the ; at the end we're
telling the compiler return the value of the expression, much like how
the last expression of a `do` block in Haskell returns a value.

Pretty simple right? The only difference between FFI code and Rust only code
is the `extern "C"` and `#[no_mangle]` statements put into the function
header.

DO THE THING WEHRE I SET UP A STRING EXAMPLE

All right so we have our two functions we want to use in Haskell setup.
I want to talk about what we're going to be doing with the Haskell code
and some problems I ran into before actually writing this code.

### A small digression
While we could do this in one file we're going to use two of them. One
containing the foreign functions imported from Rust and our main file that will
call them from the other module. Why? Because there's no documentation
about using foreign functions imported in one module to be used in
another and it was the most frustrating things to have to deal with. To
digress for a bit, I found the documentation for Rust FFI was alright but
not the best. I could figure out what I needed but it's out of date since
it lacks any examples with dylib/cdylib and using that in other
languages. It also assumes I only want to write and use it in C code. I don't
want to use C. That's why I use Rust. I want that embedded in other languages
not the other way around.

Haskell's documentation however was messy, disorganized, scattered across
various web pages and sites, and could use a lot of love in getting fixed up.
There's no good tutorials on dynamic vs statically linking libraries and how to
do it. It just said cabal and GHC manuals covered it without linking the relevant
pages. That's honestly pretty frustrating, it felt like saying that's an exercise
left to the reader with no basic knowledge to get started. Linking to it would have
saved me many hours of digging through the documentation. There's no well written
explanations of doing FFI. The information on how to do it is there, it's just
not good or easy to parse without time and effort. This is one of the times where,
no the types really can't speak for themselves, because it's not types
I'm dealing with. It's frustrating because I really do like Haskell as a language
but the lack of good documentation really hurts the language.

Alright. Enough about bad documentation. Let's actually code what
I spent forever figuring it out so you don't have to be as frustrated as
I was, and so I can contribute rather than just being angry about it.

### Setting up our Haskell code
We're going to create two files in our src directory. Main.hs and FLib.hs. Here's what FLib.hs
should look like:

```haskell
module FLib where

import Foreign.C.Types

foreign import ccall "double_input" doubleInput :: CInt -> CInt

-- Put string example her
```

Here's what Main.hs looks like:

```haskell
import FLib

main :: IO ()
main = do
  putStrLn $ show $ doubleInput 3 --This will print 6

```

Now we need to setup our cabal file so that this actually can work:

```cabal
name:                haskellrs
version:             0.1.0.0
build-type:          Simple
cabal-version:       >=1.10

executable haskellrs-exec
  main-is:             Main.hs
  hs-source-dirs:      src
  build-depends:       base >= 4.7 && < 5
  default-language:    Haskell2010
  other-modules:       FLib
  extra-libraries:     rusty
  extra-lib-dirs:      target/release

library
  hs-source-dirs:      src
  exposed-modules:     FLib
  other-extensions:    ForeignFunctionInterface
  build-depends:       base >= 4.7 && < 5
  default-language:    Haskell2010
  extra-libraries:     rusty
  extra-lib-dirs:      target/release
```

Pretty simple file right? Before we dive into the Haskell code let's get
this example working so you can see for yourself:

```bash
cargo build --release
cabal run
```

You should see the following printed out on your screen:
```bash
6
```

You just ran Rust code inside of Haskell! Let's dissect our Haskell
code, so that we can understand how all of this works.







### Conclusion
If you want to take a look at the code I've put it up in a repository
[here](https://github.com/mgattozzi/haskellrs).
