---
layout: post
title: "2021: A Year in Review"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## Theseus's First Full Year 

Although 2021 not the first year of Theseus development, it _was_ the first whole year in which:
1. Theseus was fully open-sourced and publically known to the community.
2. Theseus received interest from academic and industry collaborators.
3. Theseus received funding for open-source development from industry (yay!).
4. Our focus shifted from prototyping research concepts to feature completeness, stabilization, and legacy compatibility.

Thanks to all the folks who contributed, advised, and interacted with myself and the rest of the Theseus team this year! 

We had an explosion of interest on GitHub:
![theseus-github-screenshot](/images/2022-posts/theseus-github-sshot.png)

And official funding from Futurewei to continue Theseus development!
![Futurewei plus Theseus collab](/images/2022-posts/futurewei_plus_theseus.png)



> We look forward to another productive year!
>
> Time to get Theseus onto some real devices! <sup>hint hint</sup>

#### Recap: major new developments
* Ability to build out-of-tree crates against Theseus using `theseus_cargo`
    * A novel extension of cargo to support building against prebuilt dependencies 
* Basic WASM execution (using `wasmi` interpreter) with core WASI support
    * Coming soon!
* Continuing efforts to support headless, no-graphics operation
* General legacy compatibility improvements, including a fuller libc implementation
* Generic, dynamic, and arbitrary thread-local storage (TLS), plus a Rust-like `thread_local!()` macro
* Tons of documentation, plus auto-published source docs and book docs!
* A full redesign of ergonomic and composable traits for device I/O, plus FAT FS support
* Performance and ergonomics improvements to the page and frame allocators
* Deferred interrupt handling tasks for better device driver performance and system interactivity  
* and many more!



### Thanks to 2021's Contributors!

Beyond our usual contributors, we had several newcomers from both Yale University and the open-source community at large who generously devoted their time to make some excellent improvements to Theseus. 
Our sincere thanks to:
 * Futurewei Technologies, especially [Sid Askary](https://www.linkedin.com/in/sid-askary-21a962) and [Yong He](https://www.linkedin.com/in/yong-he-1334902), for generously offering technical advice and funding for Theseus development.
 * [@apogeeoak](https://github.com/apogeeoak), who improved documentation quality and implemented GitHub workflows to autogenerate docs.
 * [Vikram Mullick](https://github.com/vikrammullick), who began and nearly finished support for running WASM+WASI binaries atop Theseus.
 * [Jacob Earle](https://github.com/jacob-earle), who began support for logging output on ARM microcontrollers and a pseudo-real time scheduling algorithm for Theseus.
 * [Josh Triplett](https://github.com/joshtriplett), who served as a valuable font of advice and sounding board for some of my wild Theseus ideas.
 * [Philipp Oppermann](https://github.com/phil-opp), whose project [Blog OS](https://os.phil-opp.com/) helped kickstart Theseus development a few years ago.
