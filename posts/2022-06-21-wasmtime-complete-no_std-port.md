---
layout: post
title: "Porting Wasmtime to no_std atop Theseus"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Bringing Wasmtime to `no_std`*

🎉🎉 Wasmtime now builds and runs on Theseus! 🎉🎉

This is the first port of Wasmtime to a `no_std`* environment, to the best of our knowledge, despite it being a [very](https://github.com/bytecodealliance/wasmtime/issues/75) [hot](https://github.com/bytecodealliance/wasmtime/pull/2024) [topic](https://github.com/bytecodealliance/wasmtime/pull/554) [of](https://github.com/bytecodealliance/wasmtime/issues/3495) [conversation](https://github.com/bytecodealliance/wasmtime/issues/3451).
We completed the initial minimal version of Wasmtime and [successfully ran it on Theseus on May 17th, 2022](https://github.com/kevinaboos/Theseus/commit/39a647581fdb7f259559400b6222613e3f914916).

This milestone marks the culmination of a very long journey, as evidenced by our previous several posts on this topic 
and the fact that our initial port started with a version of Wasmtime from mid-October 2021. Yeesh!
* [November 2021: Introducing the goal of running WASM on Theseus](../../../2021/11/01/October-Update-WASM.html)
* [December 2021: Porting the low-level "simple" Wasmtime crates and dependencies](../../../2021/12/31/November-December-Update-WASM.html#progress-on-porting-wasmtime-to-theseus)
* [February 2022: Porting the largest crate, `wasmtime-runtime`, Part 1](../../../2022/02/03/wasmtime-progress-update.html)
* [April 2022: Porting `wasmtime-runtime`, Part 2](../../../2022/04/12/wasmtime-progress-update-2.html)
* This post: Porting the top-level `wasmtime` crate and tying everything together

In this post, we aim to
1. Provide a high-level overview of Wasmtime and its multi-crate architecture,
2. Enumerate the changes needed to build Wasmtime on `no_std`*, and
3. Itemize the dependencies/functionality that Wasmtime requires from the underlying platform.

While we don't intend this to be a step-by-step porting tutorial, we do hope it makes it easier to port and run Wasmtime on other platforms in the future. 

> **Why is there an asterisk** by `no_std`? 
> 
> Theseus doesn't yet support the `std` library, so this is a legitimate `no_std` port.
> However, one cannot simply take this port and run it within any other `no_std` environment,
> because Wasmtime still relies on lots of functionality from the underlying platform.
> 
> We did have to add some `std`-like components to Theseus in order to satisfy Wasmtime's needs, as shown below.


"C'mon, just skip to the <s>recipe</s> code already!"

No problem, [here is the full changeset to Wasmtime](https://github.com/theseus-os/wasmtime/compare/35cdd53989b5eaa01691aac915d60cf609776ab6..c05b37c41b363008b9ff84b3493ea6d4f067cf88). 
Note that this doesn't include the many changes and extensions we made to Theseus to support this; those are described in the rest of the article. 


## 1. Summary of Wasmtime's key parts

As described in [Wasmtime's documentation](https://docs.wasmtime.dev/contributing-architecture.html), the project is architected as one top-level user-facing crate called `wasmtime` that re-exports and connects together key functionality from several internal crates.


* `wasmtime-cli`: a CLI app that offers simple interactive access to standard Wasmtime features 
* `wasmtime`: exposes a safe, embeddable API for interacting with WASM modules, e.g., compiling, instantiating, and invoking them
* `wasmtime-jit`: facilitates JIT compilation and execution of WASM modules using a compiler's code generator (currently cranelift)
* `wasmtime-runtime`: implements the majority of the runtime logic for executing WASM binaries atop of a give host platform
* `wasmtime-environ`: standalone abstract definitions of core WASM concepts and environment types, enabling integration with the cranelift backend
* `wasmtime-types`: definitions for core WASM types and execution concepts
* `wasmparser`: an external (non-Wasmtime) tool for parsing WASM binaries
<!-- * `cranelift-entity`: core data structures used by the Cranelift code generator   -->


The diagram below depicts the above major components of Wasmtime[^1] and their dependencies, with a focus on those that *did not already support `no_std`* when our work began or required other forms of modification.
This intentionally excludes ubiquitous dependencies like error handling, heap allocation, and logging to keep the graph legible (...ish).
Note that the project also contains many other crates that realize optional features, such as module caching, fibers, etc, but these (and `wasmtime-cli`) aren't necessary for an initial port. 


TODO: use better formatting and coloring for the nodes/edges of this diagram. Also, create a key/legend for what the arrow and nodes and subgraphs represent.


<a href="/images/2022-posts/wasmtime_diagram.svg" alt="Wasmtime architecture diagram with dependencies" target="_blank">
  <img align="center" src="/images/2022-posts/wasmtime_diagram.svg"/>
</a>

<div align="center"><em>(click the diagram to open it in a full-size window)</em></div>

[^1]: For simplicity, we depict `wasmparser` as part of the set of Wasmtime crates, even though it is actually part of the separate `wasm-tools` project.


## 2. Changes made to Wasmtime components

We took a bottom-up approach to iteratively port lower-level dependencies until they compiled on `no_std` on Theseus, and then moved on to the next highest layer in the dependency stack. 
The lowest-level crates, `wasmparser`, `wasmtime-types`, and `wasmtime-environ` were relatively simple to port to `no_std`, requiring only trivial changes [described in the section below](#trivial-yet-tedious-changes-for-nostd-compatibility).

By far, the most complex crate to port was `wasmtime-runtime`, which has not only myriad dependencies that each need porting, but also plenty of platform-specific code that is necessarily unsafe and quite intricate. 
We have already described our efforts to port `wasmtime-runtime` to Theseus in [two previous posts starting here](../../../2022/02/03/wasmtime-progress-update.html).

Moving up, porting `wasmtime-jit` and then `wasmtime` was fairly straightforward once `wasmtime-runtime` was done, as they share many dependencies.
There were a few issues that needed fixing, as described below, but nothing strenuous.

Finally, the real fun begins once all Wasmtime crates are able to be compiled for your platform! 
You then attempt to run it for the first time only to discover that nothing works and you don't know why.

Welcome to integration hell! 😈

... Ok, time to come clean; it turns out that this wasn't so bad. We only had a few days of debugging struggles that centered around differences in how `bincode` performs (de)serialization across different versions and how `region` handles memory management on Unix vs. Theseus. Both are explained in the following sections.

### Logic and structural changes

As Wasmtime is cross-platform, it contains several platform-specific modules that we must implement for Theseus. 
These typically look something like:
```rust
if #[cfg(unix)] {
    mod systemv;
    pub use self::systemv::*;
} else if #[cfg(target_os = "theseus")] {
    mod theseus;
    pub use self::theseus::*;  
} else ...
```
[Here's an example of one such module](<https://github.com/theseus-os/wasmtime/blob/7879420c3c75a8514c0137928d998ef29f04b22d/crates/jit/src/unwind/theseus.rs>) in `wasmtime-jit` that handles registering unwind info for new code regions that were JIT-compiled from WASM modules.

Beyond these modules, we have encountered only a few places in which platform-agnostic code must be modified to support Theseus. Two such examples are:
* The aforementioned registration of unwind info:
  ```rust
  UnwindRegistration::new(
      text.as_ptr(),
      #[cfg(target_os = "theseus")]
      text.len(), // Theseus needs the text section's length
      unwind_info.as_ptr(),
      unwind_info.len(),
  )
  ```
  * Here, Theseus's unwinder benefits from knowing both the starting *and* ending bounds of the `.text` section(s) that a given piece of unwind info covers; it helps accelerate the search for the unwind info that corresponds to a given stack frame's caller virtual address.
  * This stems from Theseus's structure of many small crate object files all being loaded and linked at runtime, which results in many separate `.eh_frame` sections that each contain unwind info for different `.text` section address ranges.

* The representation of memory-mapped region in `wasmtime-runtime`:
  ```rust
  pub struct Mmap {
      ptr: usize,
      len: usize,
      file: Option<File>,
      #[cfg(target_os = "theseus")]
      theseus_mp: theseus_memory::MappedPages,
  }
  ```
  * A [Theseus-specific version of `Mmap::accessible_reserved()`](https://github.com/theseus-os/wasmtime/blob/c05b37c41b363008b9ff84b3493ea6d4f067cf88/crates/runtime/src/mmap.rs#L172-L196) and other related functions were also necessary to accommodate the changed `Mmap` struct.
  * This design choice avoids unsafe or leaky code that would be required to get around the [ownership-based type invariants guaranteed by `MappedPages`](https://www.theseus-os.com/Theseus/doc/memory/struct.MappedPages.html).


Naturally, we made many other changes, but those were generally much smaller in scope and described in following separate sections.
The inquisitive reader can search [the full list of changes](https://github.com/theseus-os/wasmtime/compare/35cdd53989b5eaa01691aac915d60cf609776ab6..c05b37c41b363008b9ff84b3493ea6d4f067cf88) for `target_os = "theseus to see other platform-specific modifications.


### Minor yet **tedious** changes for `no_std` compatibility

Many other changes are trivial and thus glossed over in this post, such as changing import statements from `std::xyz` to `core::xyz` or `alloc::xyz` and modifying or patching dependency chains to disable `default-features`.
However, I don't want to understate the pure tedium of such changes because they require a lot of effort for no actual difference in functionality.
Many of the code diffs look like the collage of boring tedium below:
```patch
- use std::borrow::Borrow;
- use std::marker::PhantomData;
- use std::mem;
+ use core::borrow::Borrow;
+ use core::marker::PhantomData;
+ use core::mem;
```

```patch
[dependencies]
- anyhow = "1.0"
+ anyhow = { version = "1.0", default-features = false }
  cranelift-entity = { path = "../../cranelift/entity", version = "0.77.0" }
- wasmtime-types = { path = "../types", version = "0.30.0" }
+ wasmtime-types = { path = "../types", default-features = false, version = "0.30.0" }
- wasmparser = "0.81"
+ wasmparser = { version = "0.81", default-features = false }
  indexmap = { version = "1.0.2", features = ["serde-1"] }
- thiserror = "1.0.4"
+ thiserror = { version = "1.0.4", optional = true }
- serde = { version = "1.0.94", features = ["derive"] }
+ serde = { version = "1.0.94", default-features = false, features = ["derive"] }
  object = { version = "0.27", default-features = false, features = ['read_core', 'write_core', 'elf'] }
+ hashbrown = { version = "0.11.2", features = ["ahash"], default-features = false }
  ...

+ [patch.crates-io]
+ wasmparser = { path = "./path/to/ported/wasmparser/crate" }
+ object = { path = "./path/to/ported/object/crate" }
```

... and the procedure to patch dependencies that have been ported to `no_std` is even more tedious, shown below:
![porting_dependencies_to_no_std](/images/2022-posts/porting_deps_to_no_std.svg)


It would be a ***fantastic*** quality-of-life improvement if `std`'s re-exports of items from `core` and `alloc` could be transparently converted to their `no_std` equivalents, but that's a topic for another day.
In fact I've already suggested this in a presentation to select Rust team members entitled ["How Theseus uses Rust, plus Rust challenges", slides 47-51](https://www.theseus-os.com/Theseus/book/misc/papers_presentations.html#selected-presentations-and-slide-decks).


Here are a few specific examples of changes that were conceptually minor yet tricky or tedious:
* Error handling, particularly usage of `anyhow`
  * This boils down to error type mapping: `.map_err(anyhow::Error::msg)`.
* Substituting `no_std` versions of specific crates
  * `std::io` → `core2::io`
  * `thiserror` → `thiserror_core2`
* Using `bincode` for (de)serialization
    * In accordance with [`bincode`'s migration guide](https://github.com/bincode-org/bincode/blob/trunk/docs/migration_guide.md), we migrated to version 2.0 that supports `no_std`, typically like so: 
  ```diff
  - bincode::deserialize(...)
  + bincode::serde::decode_from_slice(..., bincode::config::legacy())
  ```
  * Side note: be sure to use the same versions of all types when you serialize and deserialize things... otherwise you'll get those lovely mysterious errors and spend a whole two days trying to figure out which struct field is causing the problem. 🙄



## 3. Full list of functionality needed to support Wasmtime
* Heap allocation for `alloc` types
  * Sorry, you ain't gonna run Wasmtime without a heap!
* Basic memory management: allocating memory regions, creating memory mappings, etc.
  * Unix-like OSes leverage crates like `libc`, `region`, `rsix` for this
* Stack unwinding for proper panic handling and ensuring drop handlers run
  * `wasmtime-runtime` requires both [`catch_unwind()`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) and [`resume_unwind()`](https://doc.rust-lang.org/std/panic/fn.resume_unwind.html)
  * Also need to support registration of external unwind info that comes from WASM modules compiled into native code, plus an unwinder that can utilize this info
* `backtrace` for capturing a stack trace
  * Plus optional symbolication of addresses, i.e., `addr2line` functionality or similar
* Signal handling for trapping between WASM and native execution contexts 
  * On Theseus, this means registering custom handlers for CPU exceptions (see [prior post](../../../2022/04/12/wasmtime-progress-update-2.html#the-last-sjedis-dependency-signal-handling))
* Editing object files
  * We contributed a PR to `object` that allowed reading _and_ writing of object files in a `no_std` environment
* `bincode` + `serde` for (de)serialization
  * Just had to upgrade to newer major version that already supported `no_std`
  * Required some minor changes to how `wasmtime` uses bincode, in accordance with [`bincode`'s migration guide](https://github.com/bincode-org/bincode/blob/trunk/docs/migration_guide.md)
* Basic support for accessing filesystem paths and reading files
  * Only needed when reading a AOT pre-compiled WASM module from a file for purposes of deserialization
* Multitasking
  * Full preemption is essentially required, as cooperative multitasking would require a significant rework of Wasmtime
* Thread-Local Storage (TLS), see [prior post](../../../2022/02/03/wasmtime-progress-update.html#supporting-thread-local-storage-on-theseus)
  * TLS via the `thread_local!()` macro is used to statefully handle traps and support stack unwinding during execution of native WASM module code
* Stack access
  * Typically via the `psm` crate, for portable stack manipulation
  * Only used to obtain the current value of the stack pointer register
* I/O traits
  * Mostly just the basic ones from `std::io`, which are usable in `core` through a variety of `no_std` crates like `core2` (previously `bare_io`), `core_io`, etc
* Basic error handling via `anyhow` and `thiserror`
  * `anyhow` already supports `no_std`, but has a different API due to its reliance on `std::error::Error`
  * `thiserror` cannot support `no_std` yet, but there are `no_std`-compatible drop-in alternatives like `thiserror_core2` which derive `core2::error::Error` instead
  * Once the `std::error::Error` trait is moved into `core`, all of these pain points will be permanently healed!
* A source of randomness
  * Easiest option is to use OS-provided RNG via `rand`'s interface, or its `no_std` `SmallRng` feature
* Basic I/O for printing output to a screen, console, or log
* Mutual exclusion
  * The `spin` crate provides an easy yet inefficient `no_std` spinlock, but we do prefer to use Theseus's own `Mutex` that blocks the current task by putting it to sleep
* `crc32fast`, a dependency of the `object` crate
  * We contributed a PR to make this compile properly for custom targets
* `more_asserts` for easy assertions
  * We contributed a trivial PR to make this `no_std`




## Concluding remarks + next steps

Phew, that was a long ride — but it's only the beginning of the journey to bring Wasmtime to Theseus.
We still have several optional features to port and implement, starting with full support for live JIT compilation of WASM modules.
We then plan to integrate Theseus's [existing WASI implementation](https://github.com/theseus-os/Theseus/blob/0e61ea815cbf2dc4857b8e484a5f39587e6bc25a/kernel/wasi_interpreter/src/wasi_syscalls.rs) with our port of Wasmtime such that we can more easily run legacy programs in a WASM environment.

This is a major addition to a [safe-language OS](https://www.theseus-os.com/Theseus/book/design/idea.html) like Theseus, which runs everything in a single address space and single privilege level, foregoing hardware protection in favor of reliance on language-provided safety to realize isolation between both applications and system components.
Importantly, this will enable Theseus to safely execute unsafe code or legacy components written in unsafe languages, using WASM as a "sandbox" to prevent unsafe code from circumventing the type-based restrictions  ensured and obeyed by the rest of the OS's safe Rust environment.

Further down the line, we would like to propose design changes to Wasmtime such that it makes greater usage of traits to abstract away its platform-specific dependencies as much as possible.
This would make it easier to port Wasmtime to new platforms, and ideally reduce the amount of mandatory unsafe code.

### Small demo

Given the following very simple WASM module `hello.wat`:
```wasm
(module
    (import "host" "hello" (func $host_hello (param i32)))
    (func (export "hello")
        i32.const 3
        call $host_hello)
)
```

we can precompile it using the `wasmtime-cli` on Linux or another machine:
```sh
wasmtime --compile --target x86_64-theseus hello.was --output hello.cwasm
```

and then run that precompiled `hello.cwasm` binary using the following host code on Theseus:
```rust
use wasmtime::{Caller, Engine, Func, Instance, Module, Store};

/// Taken from the docs example code in `wasmtime/crates/wasmtime/src/lib.rs`.
fn hello(cwasm_file_contents: &[u8]) -> Result<...> {
    let engine = Engine::default();
    // Deserialize the AOT pre-compiled WASM binary
    let module = unsafe {
        Module::deserialize(&engine, cwasm_file_contents)? 
    };
    let mut store = Store::new(&engine, 4);
    let host_hello = Func::wrap(&mut store, |caller: Caller<'_, u32>, param: i32| {
        println!("Got {} from WebAssembly", param);
        println!("my host state is: {}", caller.data());
    });
    let instance = Instance::new(&mut store, &module, &[host_hello.into()])?;
    let hello = instance.get_typed_func::<(), (), _>(&mut store, "hello")?;
    // Invoke the WASM `hello` function
    hello.call(&mut store, ()).map_err(anyhow::Error::msg)?;
    ...
}
```

If all goes according to plan, you'll see the expected output below:

<img align="left" src="/images/2022-posts/test_wasmtime_screenshot.png" alt="wasmtime on Theseus screenshot" width="500"/>

<br clear="all">