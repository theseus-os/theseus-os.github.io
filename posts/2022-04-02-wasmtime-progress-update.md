---
layout: post
title: "Spring 2022 Update: `wasmtime-runtime` Progress"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## The ~quest~ port continues!

TODO: topics to cover: from Feb 1 to April 1

* Support backtrace, which required modifying Theseus's unwinder and stack trace functionality to expose more details about what its doing and the register values on each stack frame
* Support `std`-like `path`s in Theseus's `no_std` environment
  * Plus syntactic sugar to use a regular `Rust` `String` and `str` as an `OsString` and `OsStr`, respectively
* Improved unwinding, now supports `resume_unwind`
* Better error traits and error handling for `no_std`, including adaptations of the ubiquitous `thiserror` and `anyhow`





### The Last ~Jedi~ Dependency
* Signal handling
  * Required because `wasmtime` performs Just-In Time (JIT) compilation of WASM binaries into native code, so a failure or exception will translate into a native host exception, e.g., a SEGFAULT or the like.



## Miscellaneous Contributions
* Jacob's rate monotonic scheduler for pseudo-realtime tasks
  * Introduce a `sleep` crate that handles blocking a task for a given number of system ticks.
  * Supports the notion of "periodic" tasks that run a short workload, go to sleep, and then wake back up at the beginning of every X-tick period.
* Updated to the latest Rust 1.61 nightly version.
  * Required to fix the ABI issues in LLVM, wherein the `"x86-interrupt"` ABI for exception handlers didn't properly match what the CPU pushes onto the stack before jumping to said handler.
* Contributed a tiny PR back to the `x86_64` crate
  * We had been using our own outdated fork of this library for a while, but I minimized Theseus's dependencies on it so we can now rely on it for only a small subset of features. This made it easier to upgrade to the latest version.
* Ramla finished implementing packet tranmission for the Mellanox 100GiB NIC

