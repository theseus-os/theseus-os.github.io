---
layout: post
title: "Porting wasmtime to no_std atop Theseus"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Bringing `wasmtime` to `no_std`*

🎉🎉 `wasmtime` now builds and runs on Theseus! 🎉🎉

This is the first port of `wasmtime` to a `no_std`* environment, to the best of our knowledge, despite it being a [very](https://github.com/bytecodealliance/wasmtime/issues/75) [hot](https://github.com/bytecodealliance/wasmtime/pull/2024) [topic](https://github.com/bytecodealliance/wasmtime/pull/554) [of](https://github.com/bytecodealliance/wasmtime/issues/3495) [conversation](https://github.com/bytecodealliance/wasmtime/issues/3451).
We completed the initial minimal version of `wasmtime` and [successfully ran it on Theseus on May 17th, 2022](https://github.com/kevinaboos/Theseus/commit/39a647581fdb7f259559400b6222613e3f914916).

This milestone marks the culmination of a very long journey, as evidenced by our previous several posts on this topic 
and the fact that our initial port started with a version of `wasmtime` from mid-October 2021. Yeesh!
* [November 2021: Introducing the goal of running WASM on Theseus](../../../2021/11/01/October-Update-WASM.html)
* [December 2021: Porting the low-level "simple" `wasmtime` crates and dependencies](../../../2021/12/31/November-December-Update-WASM.html#progress-on-porting-wasmtime-to-theseus)
* [February 2022: Porting the largest crate, `wasmtime-runtime`, Part 1](../../../2022/02/03/wasmtime-progress-update.html)
* [April 2022: Porting `wasmtime-runtime`, Part 2](../../../2022/04/12/wasmtime-progress-update-2.html)
* This post: Porting the top-level `wasmtime` crate and tying everything together

In this post, we aim to
1. Provide a high-level overview of `wasmtime` and its multi-crate architecture,
2. Enumerate the changes needed to build `wasmtime` on `no_std`*, and
3. Itemize the dependencies/functionality that `wasmtime` requires from the underlying platform.

While we don't intend this to be a step-by-step porting tutorial, we do hope it makes it easier to port and run `wasmtime` on other platforms in the future. 

> **Why is there an asterisk** by `no_std`? 
> 
> Theseus doesn't yet support the `std` library, so this is a legitimate `no_std` port.
> However, one cannot simply take this port and run it within any other `no_std` environment,
> because `wasmtime` still relies on lots of functionality from the underlying platform.
> 
> We did have to add some `std`-like components to Theseus in order to satisfy `wasmtime`'s needs, as shown below.


"C'mon, just skip to the <s>recipe</s> code already!"

No problem, [here is the full changeset to `wasmtime`](https://github.com/theseus-os/wasmtime/compare/35cdd53989b5eaa01691aac915d60cf609776ab6..c05b37c41b363008b9ff84b3493ea6d4f067cf88). 
Note that this doesn't include the many changes and extensions we made to Theseus to support this; those are described in the rest of the article. 


## 1. Summary of `wasmtime`'s key parts

As described in [`wasmtime`'s documentation](https://docs.wasmtime.dev/contributing-architecture.html), the project is architected as one top-level user-facing crate called `wasmtime` that re-exports and connects together key functionality from several internal crates.


* `wasmtime-cli`: a CLI app that offers simple interactive access to standard `wasmtime` features 
* `wasmtime`: exposes a safe, embeddable API for interacting with WASM modules, e.g., compiling, instantiating, and invoking them
* `wasmtime-jit`: facilitates JIT compilation and execution of WASM modules using a compiler's code generator (currently cranelift)
* `wasmtime-runtime`: implements the majority of the runtime logic for executing WASM binaries atop of a give host platform
* `wasmtime-environ`: standalone abstract definitions of core WASM concepts and environment types, enabling integration with the cranelift backend
* `wasmtime-types`: definitions for core WASM types and execution concepts
* `wasmparser`: an external (non-`wasmtime`) tool for parsing WASM binaries
<!-- * `cranelift-entity`: core data structures used by the Cranelift code generator   -->


The diagram below depicts the above major components of `wasmtime`[^1] and their dependencies, with a focus on those that *did not already support `no_std`* when our work began or required other forms of modification.
This intentionally excludes ubiquitous dependencies like error handling, heap allocation, and logging to keep the graph legible (...ish).
Note that the project also contains many other crates that realize optional features, such as module caching, fibers, etc, but these (and `wasmtime-cli`) aren't necessary for an initial port. 


TODO: use better formatting and coloring for the nodes/edges of this diagram. Also, create a key/legend for what the arrow and nodes and subgraphs represent.


```mermaid
flowchart TB;

    top{"Top-level <br> Application"}

    top --> wasmtime    

    subgraph wasmtime_components [ ] %% ["<tt>wasmtime</tt> components"]
        wasmtime("<tt>wasmtime</tt> <br> (Theseus-specific port)")
        jit("<tt>wasmtime-jit</tt> <br> (Theseus-specific port)")
        runtime("<tt>wasmtime-runtime</tt> <br> (Theseus-specific port)")
        environ("<tt>wasmtime-environ</tt> <br> (Mostly <tt>no_std</tt>, needs <tt>Path</tt>)")
        types("<tt>wasmtime-types</tt>  <br> (true <tt>no_std</tt> port)")
        parser("<tt>wasmparser<sup>1</sup></tt> <br> (true <tt>no_std</tt> port)")
    end

    %%%% dependencies for wasmtime-jit
    wasmtime --> runtime
    wasmtime --> environ
    wasmtime --> jit
    wasmtime --> parser
    wasmtime ---> memory
    wasmtime ---> stack_trace
    wasmtime ---> object_file
    wasmtime ---> serialization
    wasmtime ---> unwinding
    wasmtime ---> file

    %%%% dependencies for wasmtime-jit
    jit --> environ
    jit --> runtime
    jit --> parser
    jit ---> memory
    jit ---> serialization
    jit ---> symbolication
    jit ---> object_file
    %% jit ----> external_unwind_info %% this unnecessarily complicates the diagram

    %%%% dependencies for wasmtime-runtime
    runtime --> environ
    runtime ------> memory
    runtime ------> signals
    runtime ------> multitasking
    runtime ------> unwinding
    runtime ------> stack_trace
    runtime ------> randomness
    runtime ------> file
    runtime ------> tls

    %%%% dependencies for wasmtime-environ
    environ --> types
    environ --> parser
    environ --> serialization
    environ --> object_file

    %%%% dependencies for wasmtime-types
    types --> parser
    types ---> serialization


    subgraph platform [ ] %% [Platform-Specific Features]
        memory[["Memory Management <br> <tt>alloc</tt> heap types, <br> <tt>mmap()</tt>, <tt>munmap()</tt>, etc"]]
        unwinding[["Stack Unwinding <br> <tt>catch_unwind()</tt>, <br> <tt>resume_unwind()</tt>"]]
        symbolication[["Symbolication <br> (à la <tt>addr2line</tt>)"]]
        stack_trace[[Stack trace]]
        signals[["Signal Handling"]]
        object_file[["Read & write <br> object files"]]
        tls[["Thread-Local Storage (TLS)"]]
        file[["File read"]]
        multitasking[["Multitasking"]]
        randomness[["Randomness"]]
        serialization[["(De)serialization"]]
    end


    subgraph third_party [ ]
        libc(("<tt>libc</tt>"))
        region(("<tt>region</tt>"))
        backtrace(("<tt>backtrace</tt>"))
        rand(("<tt>rand</tt> <br> (<tt>SmallRng</tt>)"))
        object(("<tt>object</tt>, <br> <tt>gimli</tt>"))
        serde(("<tt>serde</tt>, <br> <tt>bincode</tt>"))
    end


    backtrace ---> unwinder
    object_file <-.- object
    symbolication <-.- backtrace
    randomness <-.- rand
    stack_trace <-.- backtrace
    serialization <-.- serde


    subgraph theseus [ ]
        heap{{"<tt>multi_heap</tt> <br> per-core <tt>slab</tt> heaps"}}
        mapped_pages{{"<tt>memory::MappedPages</tt>"}}
        unwinder{{"<tt>unwinder</tt>"}}
        external_unwind_info{{"<tt>external_unwind_info</tt>"}}
        exception_handler{{"<tt>exception_handler</tt>"}}
        task{{<tt>task</tt>, <tt>spawn</tt>}}
        thread_local{{<tt>thread_local</tt>}}
        theseus_fs{{<tt>theseus_fs</tt>}}
    end

    unwinding <-..- external_unwind_info
    unwinding <-..- unwinder
    memory <-..- mapped_pages
    memory <-..- libc
    memory <-..- region
    memory <-..- heap

    libc --> mapped_pages
    region --> mapped_pages

    signals <-..- exception_handler
    multitasking <-..- task
    tls <-..- thread_local
    tls <-..- task
    file <-..- theseus_fs

```

[^1]: For simplicity, we depict `wasmparser` as part of the set of `wasmtime` crates, even though it is actually part of the separate `wasm-tools` project.


## 2. Changes made to `wasmtime` components

We took a bottom-up approach to iteratively port lower-level dependencies until they compiled on `no_std` on Theseus, and then moved on to the next highest layer in the dependency stack. 
The lowest-level crates, `wasmparser`, `wasmtime-types`, and `wasmtime-environ` were relatively simple to port to `no_std`, requiring only trivial changes [described in the section below](#trivial-yet-tedious-changes-for-nostd-compatibility).

By far, the most complex crate to port was `wasmtime-runtime`, which has not only myriad dependencies that each need porting, but also plenty of platform-specific code that is necessarily unsafe and quite intricate. 
We have already described our efforts to port `wasmtime-runtime` to Theseus in [two previous posts starting here](../../../2022/02/03/wasmtime-progress-update.html).

Moving up, porting `wasmtime-jit` and then `wasmtime` was fairly straightforward once `wasmtime-runtime` was done, as they share many dependencies.
There were a few issues that needed fixing, as described below, but nothing strenuous.

Finally, the real fun begins once all `wasmtime` crates are able to be compiled for your platform! 
You then attempt to run it for the first time only to discover that nothing works and you don't know why.

Welcome to integration hell! 😈

... Ok, time to come clean; it turns out that this wasn't so bad. We only had a few days of debugging struggles that centered around differences in how `bincode` performs (de)serialization across different versions and how `region` handles memory management on Unix vs. Theseus. Both are explained in the following sections.

### Logic and structural changes

As `wasmtime` is cross-platform, it contains several platform-specific modules that we must implement for Theseus. 
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
  * A [Theseus-specific version of `Mmap::accessible_reserved()`](https://github.com/theseus-os/wasmtime/blob/c05b37c41b363008b9ff84b3493ea6d4f067cf88/crates/runtime/src/mmap.rs#L172-L196) and other related functions was also necessary to accommodate the changed `Mmap` struct.
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



## 3. Full list of functionality needed to support `wasmtime`
* Heap allocation for `alloc` types
  * Sorry, you ain't gonna run `wasmtime` without a heap!
* Stack unwinding for proper panic handling and ensuring drop handlers run
  * `wasmtime-runtime` requires both [`catch_unwind()`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) and [`resume_unwind()`](https://doc.rust-lang.org/std/panic/fn.resume_unwind.html)
  * Also need to support registration of external unwind info that comes from WASM modules compiled into native code, plus an unwinder that can utilize this info
* `backtrace` for capturing a stack trace
  * Plus optional symbolication of addresses, i.e., `addr2line` functionality or similar
* Basic memory management: allocating memory regions, creating memory mappings, etc.
  * Unix-like OSes leverage crates like `libc`, `region`, `rsix` for this
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
  * Full preemption is essentially required, as cooperative multitasking would require a significant rework of `wasmtime`
* Thread-Local Storage (TLS), see [prior post](../../../2022/02/03/wasmtime-progress-update.html#supporting-thread-local-storage-on-theseus)
  * TLS via the `thread_local!()` macro is used to statefully handle traps and support stack unwinding during execution of native WASM module code
* Access to the stack
  * Only used to obtain the current value of the stack pointer register
  * Typically via the `psm` crate, for portable stack manipulation
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

TODO: insert a screenshot, the basic WASM module code, and the code on the host that runs it.

TODO: mention fleshing out the rest of `wasmtime` and supporting optional features.

TODO: mention proposing design changes to `wasmtime` such that it uses traits to abstract away its platform-specific dependencies as much as possible.