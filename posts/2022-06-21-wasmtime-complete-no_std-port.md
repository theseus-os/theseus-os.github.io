---
layout: post
title: "Porting wasmtime to no_std atop Theseus"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Bringing `wasmtime` to `no_std`*

🎉🎉 `wasmtime` now builds and runs on Theseus! 🎉🎉

This is the first port of `wasmtime` to a `no_std`* environment, to the best of our knowledge. 
The initial barebones version of `wasmtime` was [completed and successfully run on Theseus on May 17th, 2022](https://github.com/kevinaboos/Theseus/commit/39a647581fdb7f259559400b6222613e3f914916).

This milestone marks the culmination of a very long journey, as evidenced by our previous several posts on this topic 
and the fact that our initial port started with a version of `wasmtime` from mid-October 2021. Yeesh!

In this post, we aim to
1. Provide a high-level description of `wasmtime` and its multi-crate architecture,
2. Enumerate the changes needed to build `wasmtime` on `no_std`*, and
3. Itemize the dependencies/functionality that `wasmtime` requires from the underlying platform.

While we don't intend this to be a porting tutorial, we do hope it makes it easier to port and run `wasmtime` on other platforms in the future. 

> Why is there an asterisk by `no_std`? 
> 
> Theseus doesn't yet support the `std` library, so this is a legitimate `no_std` port.
> However, one cannot simply take this port and run it within any other `no_std` environment,
> because `wasmtime` still relies on basic functionality from the underlying platform.
> 
> We did have to add limited `std`-like components to Theseus in order to support `wasmtime`'s needs;
> we discuss which components were needed in the following sections. 


"C'mon, just let me skip to the <s>recipe</s> code already!"

No problem, [here is the full changeset to `wasmtime`](). Note that this doesn't include the many changes and extensions we made to Theseus to support this; those are described in the rest of the article. 


## Summary of `wasmtime`'s key parts

TODO: insert diagram of wasmtime dependencies 

NOTE: this diagram includes only the components of `wasmtime` and its dependencies that *do not already support `no_std`*. 

## A Bottom-up Approach to Porting

We take a bottom-up approach to iteratively port lower-level dependencies  


### Picking up where we left off
In our [prior post](2022/04/12/wasmtime-progress-update-2.html), we had completed

## Concluding remarks + next steps

