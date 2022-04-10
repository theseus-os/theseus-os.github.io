---
layout: post
title: "January 2022 Update: WASM has landed!"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Running WASM + WASI modules in a safe OS kernel

🚀🚀 WASM has now landed in Theseus! 🚀🚀

With [PR #472](https://github.com/theseus-os/Theseus/commit/f4aa715f0fc706a0e3b0f3f21057338c0b295ffb) being merged in, Theseus's main branch allows you to execute WASM modules (as precompiled WASM binaries) that use both the core WASM specification and basic WASI extensions. 

See our [previous blog post](2021-12-31-November-December-Update-WASM.md) for more about this effort. 

### How to use and run WASM on Theseus

Theseus doesn't yet support running an actual compiler, so you first need to download a pre-compiled WASM binary on the Theseus command line or package one up into the OS image itself during build time. We describe the latter approach below.

Theseus's build tooling now supports including arbitrary "extra files" in the generated OS `.iso` image; [click here to read more](https://github.com/theseus-os/Theseus/tree/theseus_main/extra_files) about that feature.
All you have to do is put whatever files you want included into the `extra_files/` directory in the source code repository.
Presto-magic-change-o, these extra files will be automatically loaded into Theseus's in-memory filesystem upon boot, in the `/extra_files` directory.

Once you have a WASM binary, running it is easy with the `wasm` command:
```sh
wasm /path/to/wasm/binary [args]
```

For example, try out the `exorbitant` interactive calculator like so:
```sh
wasm /extra_files/wasm/exorbitant.wasm
```

![exorbitant demo](/images/2022-posts/exorbitant-wasm.png)


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

