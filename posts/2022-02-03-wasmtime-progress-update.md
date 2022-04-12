---
layout: post
title: "Progress porting `wasmtime-runtime` to Theseus"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---


## Porting `wasmtime-runtime`, the key `wasmtime` crate
A [previous post from late 2021](2021-12-31-November-December-Update-WASM.md) chronicled the journey to port `wasmtime` to Theseus. 
While our bottom-up approach got off to a strong start, we quickly encountered difficulty when examining the `wasmtime-runtime` crate, as it contains many dependencies on platform-specific and legacy system interfaces:
* Unix-like memory mapping and protection
* Signal/trap handling 
* Thread-local storage
* Stack introspection and backtracing
* File and I/O abstractions 
* Exception (panic) handling and unwinding resumption 

This post describes our progress over a few weeks of working to add these features to Theseus in order to support `wasmtime-runtime`'s many complex dependencies.


### Porting & reorganizing third-party libraries

TODO: the rest of this post

Add ports folder for libc and other Theseus-specific ports of third-party crates (#468)

* Change compiler target specs to use "theseus" as the `target_os`

* Reorganize third-party crates:
    * `libs/` still contains standalone third-party crates that don't depend on Theseus.
    * `ports/` is new and contains third-party crates that have been ported to depend on Theseus-specific crates (such as those in `kernel/`).

* Finished porting `object`'s write feature to no_std, PR in progress.

* Finished porting `region` to Theseus, which offers platform-agnostic abstractions for virtual memory-related operations, e.g., mlock, mprotect, etc.



TODO: describe thread-local storage





## More libc support + better `theseus_cargo`
As part of the efforts to port `wasmtime` to Theseus, more improvements were made to Theseus's support for legacy interfaces like libc. The `wasmtime` runtime depends on libc-defined functions and types quite heavily, e.g., `mmap`, signal handling, and more.

-----------------------------
Finally got libc and tlibc to work together with external rust crates in a full build
Can run both C tests and Rust tests for libc + tlibc
Basic memory mapping functions (mmap) and printing (printf) are working properly
Major improvements to the theseus_cargo tool 
Supports more complex builds with native dependencies (e.g., C code linked to Rust code)
Supports more crate-types beyond “lib” and “rlib”, such as “staticlib”
---------------------------------


Improved the page allocator to allow it to merge contiguous freed chunks 
 --- Needed for loading larger C files at a fixed address
Need to work on creating position-independent executables
  --- Loading multiple C executables always at a fixed address (0x400000) cannot possibly work


Finally fixed the issues with tlibc, libc, C code, and theseus_cargo builds all working together properly. Everything now works as expected!
-----------------------------------




## Miscellaneous Contributions
* Theseus now has support for arbitrary [Thread-Local Storage (TLS) areas](https://github.com/theseus-os/Theseus/commit/3e6c50a0a45560057c6a35db4ce220760e362962), as it was needed for `wasmtime-runtime`. It's also quite useful in many other contexts.
    * Theseus also supports [TLS defined in application and kernel crates loaded at runtime](https://github.com/theseus-os/Theseus/commit/fd1f11d99f1b4252abb35fffe7ae45f9f7ca616b).
    * We also ported [Rust's `thread_local!()` macro](https://github.com/theseus-os/Theseus/commit/dd62aff423e1deceed503e3341827ba1e04dce79), which offers lazy dynamic init and cleanup of TLS areas.
