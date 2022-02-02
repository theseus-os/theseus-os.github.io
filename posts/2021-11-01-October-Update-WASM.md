---
layout: post
title: "[Draft] October 2021 Update: All about WASM"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Bringing the Wonderful World of WASM to Theseus

Two concurrent projects to support WASM on Theseus:
 1. The simple approach: use the `wasmi` intepreter crate from parity-tech 
    * Relatively simple, as `wasmi` is `no_std`-compliant and requires only minimal interfacing with the host platform in order to use it
    * Implement WASI system calls as needed, which is the glue between the WASM environment and the rest of Theseus
 2. The complex approach: port the `wasmtime` WASM runtime project to Theseus
    * Massively complex with dozens of platform-specific logic and API calls
    * Tons of legacy dependencies, e.g., libc- and POSIX-style memory management, signal handling, system calls, and usage of many Rust libstd features


