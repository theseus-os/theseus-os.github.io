---
layout: post
title: "November-December 2021 Update: We have WASM liftoff!"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Theseus now supports WASM + WASI

Great news -- executing WASM modules atop Theseus is now working! 

As proposed in a [previous post](../../../2021/11/01/October-Update-WASM.html), the initial implementation covers basic WASI system calls and uses the `wasmi` crate to execute WASM binaries using an intepreter.

Big thanks to [Vikram Mullick](https://github.com/vikrammullick) for leading this effort. Our initial work was also inspired by early work from [redshirt](https://github.com/tomaka/redshirt), a WASM-based proof-of-concept system that also used `wasmi`.

> Note: this feature is not yet merged into the `theseus_main` branch. It's coming soon!
> In the mean time, you can follow its progress in [PR #472](https://github.com/theseus-os/Theseus/pull/472).


### Demos of running WASM on Theseus

We have a few demo applications to show basic WASM + WASI functionality. 

First, a classic text adventure game called [Zork](https://en.wikipedia.org/wiki/Zork#Zork_and_Dungeon). This demonstrates using stdio through WASI-defined APIs.
![zork](/images/2021-posts/wzork.png)

Second, a simple version of `grep` version 2.0 compiled to WASM, which demonstrates using WASI's filesystem I/O interface.
![grep](/images/2021-posts/wgrep.png)

Once WASM support is officially in the main branch, we will offer an easier command line interface for loading and executing WASM binaries.

## Progress on Porting Wasmtime to Theseus
While the interpreted WASM project is wrapping up, the port of Wasmtime is just getting started.

As described in [Wasmtime's documentation](https://docs.wasmtime.dev/contributing-architecture.html), the project is architected as one top-level user-facing crate that re-exports and connects together key functionality from several smaller internal crates.
The top-level crate is aptly named `wasmtime`, and it primarily exposes a safe API for interacting with WASM modules, e.g., compiling, instantiating, and invoking them.

### Bottom-up approach -- off to a strong start 
We have taken a bottom-up approach such that we can iteratively port each crate to the Theseus environment, building and testing them as we go.
A diagram of the key crates that we care about and need to port is below.
![diagram of wasmtime key crates](/images/2021-posts/wasmtime-crate-structure.png)

As such, the top-level `wasmtime` crate will be the *last* one that we port to Theseus.
So far, we have been able to quickly adapt the following bottom-most crates (shown in green above) to `no_std` environments, as they are relatively standalone:
* `wasmparser`: an external (non-wasmtime) tool for parsing WASM binaries
* `wasmtime-types`: definitions for core WASM types and execution concepts
* `wasmtime-environ`: support for abstract definitions of compiler environment and features, enabling easy use of the cranelift backend for JIT
* `cranelift-entity`: core data structures used by the Cranelift code generator  


In addition, we had to modify the following crates that were dependencies of wasmtime in order to support `no_std` environments. We submitted PRs to each crate's upstream repository, some of which have already been accepted. Thanks to those authors/maintainers!
* `object`: PR to [allow `no_std` environments to *write*, not just read, object files](https://github.com/gimli-rs/object/pull/400)
* `more_asserts`: a trivial PR to [add `no_std` support](https://github.com/thomcc/rust-more-asserts/pull/6)
* `crc32fast`: a tricky fix to [compilation for targets that lack some SIMD hardware features](https://github.com/srijs/rust-crc32fast/pull/22) 
* `indexmap`: PR to [unify the API between `std` and `no_std` users](https://github.com/bluss/indexmap/pull/207) of `indexmap` data structures
    * Unfortunately, this one has been delayed due to `indexmap`'s non-standard usage of features and auto-`std` feature detection


### Looking forward in (wasm)time
The main challenge in porting Wasmtime to Theseus is porting the `wasmtime-runtime` crate, which implements the majority of the runtime logic for executing WASM binaries atop a given host platform.
This crate is massive and will likely take months to complete, due to its many dependencies and robust usage of standard POSIX-like functionality that Theseus currently lacks.

We'll start by tackling the main dependencies of `wasmtime-runtime`, most of which are based around legacy POSIX-style interfaces and traditional OS platform abstractions:
* Unix-like memory mapping and protection
* Signal/trap handling 
* Thread-local storage
* Stack introspection and backtracing
* File and I/O abstractions 
* Exception (panic) handling and unwinding resumption 


Look for updates in the next post!


## Miscellaneous Contributions
* Kevin wrote a [new book chapter](https://www.theseus-os.com/Theseus/book/subsystems/task.html) about how Theseus's task management subsystem works.
* Fixed a [rare bug](https://github.com/theseus-os/Theseus/issues/451) in the [frame allocator](https://github.com/theseus-os/Theseus/pull/456), originally identified by Ramla. Thanks to her for catching that!
* Fixed a classic off-by-one [bug](https://github.com/theseus-os/Theseus/issues/473) that caused buffered [stdio reads to drop one character](https://github.com/theseus-os/Theseus/commit/983532ac6dc6d7ccd69faf5c62f38e3b9760c4d6) when the buffer was full. Thanks to Vikram for catching this!
