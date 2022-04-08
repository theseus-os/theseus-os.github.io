---
layout: post
title: "November-December 2021 Update: We have WASM liftoff!"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Theseus now supports WASM + WASI

Good news -- support for executing WASM modules atop Theseus is now complete! 

As proposed in a [previous post](2021-11-01-October-Update-WASM.md), the initial implementation covers basic WASI system calls and uses the `wasmi` crate to execute WASM binaries in an intepreter.

Big thanks to [Vikram Mullick](https://github.com/vikrammullick) for leading this effort. Our initial work was also inspired by early work from [redshirt](https://github.com/tomaka/redshirt), a WASM-based proof-of-concept system that also used `wasmi`.


-------------------------------

> Note: the rest of this article is a work in progress.
