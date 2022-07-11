---
layout: post
title: "October 2021 Update: All about WASM"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Bringing the Wonderful World of WASM to Theseus

> Theseus's newest goal is to be a **WASM-native** system, in which a fully-featured WASM runtime can execute in a bare metal environment.

[WebAssembly](https://webassembly.org/) (WASM) is a powerful new binary instruction format that offers a "sandboxed" execution environment based on a simple machine model. 
WASM's goal is to allow code from a variety of programming languages to be easily deployed on and performantly executed within a web browser-provided environment, effectively realizing the portability dream once envisioned by Java's bytecode format.

In addition, multiple extensions to the standard help to expand WASM's functionality beyond just what is offered by most browsers.
The most notable is [WASI](https://github.com/WebAssembly/WASI), the WebAssembly System Interface, which extends WASM's core functionality with common system-provided features like standard I/O, filesystems, clocks and timekeeping, and more.

### Why WASM on Theseus?
WASM is the solution to one of the major downsides of safe-language OSes: all components must be written in a safe language in order to uphold the isolation and safety guarantees provided by said language compiler.
This can make it tedious or impossible to support legacy components and interfaces.

With a WASM runtime, Theseus could safely load and run software modules written in any arbitrary unsafe language!
All you'd need to do is compile them into a WASM module, which is quite easy thanks to most major languages supporting WASM targets.
This will also make it significantly easier to run legacy components with complex dependency chains atop Theseus, as we can bundle them all up into self-contained WASM modules with little to no external dependencies.
### How do we get there?
To bring WASM to Theseus, we have started two concurrent projects:
 1. The simple approach: use the [`wasmi` intepreter crate from parity-tech](https://github.com/paritytech/wasmi) 
    * Relatively simple, as `wasmi` is `no_std`-compliant and requires only minimal interfacing with the host platform in order to use it
    * We can implement WASI system calls as needed, which acts as the glue is the glue between the WASM environment and the rest of Theseus's subsystems
 2. The complex approach: port the [Wasmtime WASM runtime project](https://github.com/bytecodealliance/wasmtime) to Theseus
    * Massively complex with dozens of platform-specific logic and API calls
    * Tons of legacy dependencies, e.g., libc- and POSIX-style memory management, signal handling, system calls, and usage of many Rust libstd features


[Vikram Mullick](https://github.com/vikrammullick) has begun working on part 1 above as part of his senior capstone project at Yale.
[Kevin Boos](https://github.com/kevinaboos) has begun working on part 2 above, and will also assist with part 1 as needed. 

Due to the complex nature of Wasmtime with its many legacy dependencies, this two-pronged split approach is quite beneficial, giving us the best of both worlds:
* Theseus can "quickly" get up and running with basic WASM support, allowing us to:
   * Experiment with running legacy components as WASM modules
   * Begin implementing support for WASI and other key WASM interfaces, e.g., WebGL
   * Integrate Theseus's existing runtime loading and linking infrastructure with WASM
* We can leverage the existing WASM and WASI infrastructure layers to more easily support Wasmtime
   * This will realize a ~10x performance improvement over the initial `wasmi`, without wasting the initial `wasmi`-based efforts


We look forward to announcing WASM support for Theseus and realizing its full potential as a WASM-native system.
