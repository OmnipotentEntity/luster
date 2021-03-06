[![Build Status](https://travis-ci.org/kyren/luster.svg?branch=master)](https://travis-ci.org/kyren/luster)

# luster - An experimental Lua VM implemented in pure Rust #

My eventual goals with `luster` are somewhat ambitious:
  * Be a practical, useful Lua interpreter that is "pragmatically compatible"
    with the latest PUC-Rio Lua (5.3, soon 5.4)
  * Be generally at least as fast as PUC-Rio Lua
    * Using primarily safe Rust
  * Allow creating safe Lua bindings to Rust that are dramatically easier and
    faster than what is possible with `rlua` and PUC-Rio's C API.
  * Demonstrate a novel set of techniques for using garbage collected pointers
    in safe Rust, and show that the techniques work by implementing a real
    project with them.

**This project is in active development and currently very WIP**

## A novel system for Rust <-> GC interaction ##

`luster` has a real, cycle detecting, incremental garbage collector with
zero-cost `Gc` pointers (they are machine pointer sized and implement `Copy`)
and are usable from safe Rust.  It achieves this by combining three techniques:

1) An unsafe `Collect` trait which allows tracing through garbage collected
   types that, despite being unsafe, can be implemented safely using procedural
   macros.
2) Branding `Gc` pointers by unique, invariant "generative" lifetimes to ensure
   that such pointers are isolated to a single root object, and to guarantee
   that, outside an active call to `mutate`, all such pointers are either
   reachable from the root object or are safe to collect.
3) The mutation API, while being safe via "generativity", does not make it easy
   to allow garbage collection to take place continuously.  Since no garbage
   collection at all can take place during a call to `mutate`, long running
   mutations are problematic.  By using a `futures`-like combinator based
   "sequencing" API, we can recover the ability for garbage collect to take
   place with as fine of a granularity as necessary, with garbage collection
   taking place in-between the "sequence" steps.

(These ideas are not all mine, the garbage collector is heavily derived from
[rust-gc](https://manishearth.github.io/blog/2015/09/01/designing-a-gc-in-rust/),
and the idea of using "generativity" comes from [You can't spell trust without
Rust](https://raw.githubusercontent.com/Gankro/thesis/master/thesis.pdf).)

While the interface to garbage collected pointers is interesting, the actual
garbage collector itself is currently only a very basic (but adequate)
incremental mark-and-sweep collector.  This could be replaced in the future with
a better design.

## What currently works ##

* An actual cycle detecting, incremental GC similar to the one in PUC-Rio Lua
  5.3
* A basic Lua bytecode compiler
* Lua source code is compiled to a VM bytecode similar to PUC-Rio Lua's, and
  there are a complete set of VM instructions implemented (minus bitwise
  mathematical operators).
* Most of the core Lua language works (currently only missing bitwise
  mathematical operators).  Some tricky Lua features that currently work:
  * Real closures with proper upvalue handling
  * Tail calls
  * Variable arguments and returns
  * Coroutines, including yielding through Rust callbacks (like through `pcall`)
  * gotos with label handling that matches Lua 5.3
  * proper _ENV handling
* A few tiny bits of the stdlib (`print`, `error`, `pcall`, and the hard bits
  from `coroutine`)
* Basic support for Rust callbacks (missing some fast-path APIs that I think
  will be necessary).
* A simple REPL (try it with `cargo run luster`!)

## What currently doesn't work ##

* Most of the stdlib is not implemented (`debug` (which may never be completely
  implemented), `io`, `math`, `os`, `package`, `string`, `table`, `utf8`, most
  top-level functions are unimplemented.
* Metatables and metamethods.  Most of this should not be terribly hard to
  implement *except* `__gc`, which will require implementing finalizers in
  `gc-arena`.
* Garbage collector finalization.  An algorithm and basic API for finalization
  is not difficult, but I am not quite sure yet how to design an API around
  finalizers with *failure*, which is required to implement Lua `__gc`
  metamethods.
* Lua userdata.  Basic support for a `Box<Any>` userdata type is not difficult,
  but letting userdata safely participate in garbage collection and having easy,
  performant APIs for userdata methods are much harder.
* Tables with weak keys / values, "ephemeron" tables.
* The compiled VM code is in a couple of ways worse than what PUC-Rio Lua will
  generate.  Notably, there is a JMP chaining optimization that is not yet
  implemented that makes most loops much slower than in PUC-Rio Lua.
* Error messages that don't make you want to cry
* Stack traces
* Debugger
* Actual optimization and real effort towards matching PUC-Rio Lua's performance
* Probably much more that I haven't listed

## What may never be implemented ##

This is not an exhaustive list, but these are some things which I currently
consider non-goals.  This list is also preliminary, everything here is up for
discussion:

* An API compatible with the PUC-Rio Lua C API.  It would be amazingly difficult
  to implement and would be very slow, and some of it would be basically
  impossible (longjmp error handling and adjacent behavior).
* Perfect compatibility with certain classes of behavior in PUC-Rio Lua:
  * PUC-Rio Lua behaves differently on systems depending on the OS, environment,
    compilation settings, system locale (lexing numbers changes depending on the
    system locale!), etc.  `luster` is more or less aiming to emulate PUC-Rio
    Lua behavior with the "C" locale set with the default settings in
    `luaconf.h` on 64-bit Linux.
  * The specific format of error messages.
  * The specific iteration order of tables, and the specific behavior of the
    length operator (the length operator currently functions correctly and will
    always return a table "border", but for tables that are not sequences,
    the choice of border that is returned may differ).
* Some of the `debug` library may be problematic to implement (I am not
  completely sure what yet, though)
* Compatibility with PUC-Rio Lua bytecode
* `os.setlocale`
* `package.loadlib` and all functionality which allows loading C libraries.
* Being able to predictably catch `__gc` errors in Lua (I am not sure about this
  one yet, this may be difficult or it may not).

## Contributing ##

The project is still in an early state and there is lots left to do!  If you are
interested in contributing, please take a look at [TODO.md](TODO.md) for ideas.

Much of the work left to do is *design* work rather than simply implementing
features, and this is probably the place where help would be most appreciated.
Almost none of the internal APIs are what I would consider final, and for some
of the very tricky pieces like `gc-sequence` and callbacks, there is a LOT of
room for improvement in the API design (to put it mildly!).  If you think you
have a way to make using `luster` more ergonomic, or simply want to complain
about ways that it is not ergonomic (there are many), please feel free to file
an issue and we can discuss it!

## License ##

`luster` is licensed under either of:

* MIT license [LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT
* Creative Commons CC0 1.0 Universal Public Domain Dedication
  [LICENSE-CC0](LICENSE-CC0) or
  https://creativecommons.org/publicdomain/zero/1.0/

at your option.