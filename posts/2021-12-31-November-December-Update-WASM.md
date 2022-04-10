---
layout: post
title: "November-December 2021 Update: We have WASM liftoff!"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Theseus now supports WASM + WASI

Great news -- executing WASM modules atop Theseus is now working! 

As proposed in a [previous post](2021-11-01-October-Update-WASM.md), the initial implementation covers basic WASI system calls and uses the `wasmi` crate to execute WASM binaries using an intepreter.

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

## Progress on Porting `wasmtime` to Theseus
While the interpreted WASM project is wrapping up, the port of `wasmtime` is just getting started.

TODO: finish this

TODO: describe the basic architecture of all the crates in the wasmtime tree. Taking a bottom-up approach. It was relatively quick to port the initial four lowest-level crates to a `no_std` environment, but the `wasmtime-runtime` crate is a massive one that will take months to support, due to its many dependencies and robust usage of standard POSIX-like functionality that Theseus currently lacks.


TODO: describe thread-local storage


------------------------
Submitted no_std port for the more-asserts crate, was merged in by the maintainer.
Submitted a PR to the indexmap crate that unifies the downstream API across std and no_std crates
This is used widely across wasmtime crates
Submitted a PR to the object crate that supports the object file writing abilities in no_std environment
Submitted a PR to the crc32fast crate that offers fast versions of the crc32 hashing algorithm to fix its incorrect feature gating for SSE instructions
Working on porting the `region` crate to Theseus, which supports virtual memory-related operations, e.g., mlock, mprotect, etc.
---------------------------


--------------------------------------
Add ports folder for libc and other Theseus-specific ports of third-party crates (#468)

* Change compiler target specs to use "theseus" as the `target_os`

* Reorganize third-party crates:
    * `libs/` still contains standalone third-party crates that don't depend on Theseus.
    * `ports/` is new and contains third-party crates that have been ported to depend on Theseus-specific crates (such as those in `kernel/`).

* Finished porting `object`'s write feature to no_std, PR in progress.

* Finished porting `region` to Theseus.
-----------------------------

## Miscellaneous Contributions
* Kevin wrote a [new book chapter](https://www.theseus-os.com/Theseus/book/subsystems/task.html) about how Theseus's task management subsystem works.
* Fixed a [rare bug](https://github.com/theseus-os/Theseus/issues/451) in the [frame allocator](https://github.com/theseus-os/Theseus/pull/456), originally identified by Ramla. Thanks to her for catching that!
* Fixed a classic off-by-one [bug](https://github.com/theseus-os/Theseus/issues/473) that caused buffered [stdio reads to drop one character](https://github.com/theseus-os/Theseus/commit/983532ac6dc6d7ccd69faf5c62f38e3b9760c4d6) when the buffer was full. Thanks to Vikram for catching this!
