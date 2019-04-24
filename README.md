# Internment &emsp; [![Latest version](https://img.shields.io/crates/v/internment.svg)](https://crates.io/crates/internment) [![Documentation](https://docs.rs/internment/badge.svg)](https://docs.rs/internment) [![Build Status](https://travis-ci.org/droundy/internment.svg?branch=master)](https://travis-ci.org/droundy/internment) [![Windows Build status](https://ci.appveyor.com/api/projects/status/3dps5r08b5a78fyu?svg=true)](https://ci.appveyor.com/project/droundy/internment)

A very easy to use library for
[interning](https://en.wikipedia.org/wiki/String_interning)
strings or other data in rust.  Interned data is very efficient to
either hash or compare for equality (just a pointer comparison).
Data is also automatically de-duplicated.

You have three options with the internment crate:

1. `Intern`, which will never free your data.  This means that an
`Intern` is `Copy`, so you can make as many copies of the pointer
as you may care to at no cost.

2. `LocalIntern`, which will only free your data when the calling
thread exits.  This means that a `LocalIntern` is `Copy`, so you can
make as many copies of the pointer as you may care to at no cost.
However, you cannot share a `LocalIntern` with another thread.  On the
plus side, it is faster to create a `LocalIntern` than an `Intern`.

3. `ArcIntern`, which reference-counts your data and frees it when
there are no more references.  `ArcIntern` will keep memory use
down, but requires locking whenever a clone of your pointer is
made, as well as when dropping the pointer.

In each case, accessing your data is a single pointer dereference, and
the size of any internment data structure (`Intern`, `LocalIntern`, or
`ArcIntern`) is a single pointer.  In each case, you have a guarantee
that a single data value (as defined by `Eq` and `Hash`) will
correspond to a single pointer value.  This means that we can use
pointer comparison (and a pointer hash) in place of value comparisons,
which is very fast.

# Example
```rust
use internment::Intern;
let x = Intern::new("hello");
let y = Intern::new("world");
assert_ne!(x, y);
println!("The conventional greeting is '{} {}'", x, y);
```

# Comparison with other interning crates

There are already several interning crates available on
[crates.io](https://crates.io/search?q=intern).  What makes
`internment` different?  Many of the interning crates are specific to
strings.  The general purpose interning crates are:

1. [symtern](https://crates.io/crates/symtern)

2. [shawshank](https://crates.io/crates/shawshank)

3. [intern](https://crates.io/crates/intern) which is a work-in-progress.

Each of these crates implement arena allocation, with tokens of
various sizes to reference an interned object.  This approach makes
them far more challenging to use than `internment`.  Their approach
also enables freeing of all interned objects at once when they go out
of scope (which is an advantage).

The primary disadvantages of arena allocation relative to
`internment`'s approach are:

1. Lookup of a token could fail, either because an invalid token could
   be generated by hand, or a token from one pool could be used by
   another.  This adds an element of unsafety to code that uses
   interned objects:  either they assume that they are bug-free and
   panic on errors, or they have error handling any place that uses
   tokens.

2. Lookup of a token could give the wrong object, if multiple pools
   are used.  This is easy to avoid if you avoid ever using more than
   one pool, but then you may gain little benefit from the arena
   allocation.

3. Lookup of a token is slow.  They all advertise being fast, but any
   lookup is going to be slower than pointer dereferencing.  To be
   fair, increased memory locality *could* in principle make token
   lookup faster for some access patterns, but I doubt it.

To balance this, because `internment` has tokens that are globally
valid, it uses a `Mutex` to protect its internal data, which is taken
on the interning of new data as well as changing of reference counts,
which is probably slower than the other interning crates (unless you
want to use their tokens across threads, in which case you'd have to
put the pool in a `Mutex` and pay the same penalty).

Another interning crate which is very similar to `internment` is:

1. [hashconsing](https://crates.io/crates/hashconsing)

The `hashconsing` crate is considerably more complicated in its API
than `internment`, but generates global pointers in a similar way.
The `HConsed<T>` data type is always referenced counted with `Arc`,
which makes it similar to `ArcIntern`, which is less efficient than
`Intern`, but does not eternally leak memory.
