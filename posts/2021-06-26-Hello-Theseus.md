---
layout: post
title: "Hello (World!) from Theseus"
author: Kevin Boos
release: false
---

## Theseus's First Blog Post

Hello, World! 

This is the introductory post for the Theseus OS Blog, which will be used to provide more details about the changes and developmental progress of [Theseus OS](https://github.com/theseus-os/Theseus).

With this blog, we hope to achieve the following goals:
 1. Provide an easier, more transparent way to follow changes to Theseus OS than having to pore over GitHub commits
 2. Announce major changes, feature additions, and releases
 3. Share interesting tidbits related to low-level and embedded Rust development
 4. Inspire folks in the open-source community to get involved and contribute 
 5. Collect thoughts, ideas, and feedback from the Rust and OS community more directly


## The Path from Research to Usability

Since development began at Rice University a few years ago, our focus has been solely on pursuing designs with strong research merit rather than achieveing usability or feature-completeness.
As such, the current state of Theseus is a strange (im)balance between:
 * 😊 An existing set of highly-advanced system features:
   * Fully-safe memory management for arbitrary memory regions
   * Compiler-assisted resource and state management
   * Dynamic loading and linking of system components at runtime
   * Full cuustom unwinding from high-level applications to low-level kernel entities
   * Robust fault tolerance with a tiny core dependency set 
   * Live evolution of components at any layer/level of the system
 * 🙁 More "basic" features that are missing:
   * No true support for filesystems
   * Interactive shell is very minimal
   * Poor graphics support with slow compositing
   * Lacking device support beyond mouse, keyboard, and networking
   * Cannot run standard applications that use libc/libstd

**That all changes today!**

We'll now focus on proving that Theseus can be useful in real-world environments (beyond just research applications) by:
 * Analyzing, auditing, and shoring up its existing functionality
 * Fleshing out its interfaces and missing subsystems
 * Improving stability, genericness, and usability of primary subsystems
 * Working on legacy compatibility, including libc, Rust's libstd, and running WASM binaries

 That being said, researchers at Yale University will continue to use Theseus as a foundation for novel OS research, and their contributions may be featured here as well, when appropriate.

## Funding from Futurewei
Since being published in the [SOSP 2020 conference](https://www.usenix.org/conference/osdi20/presentation/boos), Theseus has garnered a lot of interest from both fellow academic institutions as well as industry researchers.
Among the interested parties was [Futurewei Technologies](https://futurewei.com/), who reached out to our research lab at Yale University (previously at Rice University) to inquire about the future of Theseus and to determine whether it could prove useful for various important domains, such as automotive computing.

Futurewei graciously offered to fund me (Kevin Boos), the creator of Theseus, to work on Theseus in a full-time capacity.
Futurewei has made a formal committment that all intellectual property and artifacts produced from work on Theseus will continue to be made open-source and remain so indefinitely, and that we will retain full control of the direction of the project. 
In addition, Futurewei has committed to funding a variety of other significant projects, teams, and individuals in the Rust community, from Rust core, compiler, and language teams themselves to folks like us working on Rust-centric projects.
Together, we're building a strong Rust ecosystem for the next generation of safe, efficient computing.


## Looking forward

I am both honored and excited to be able to continue developing Theseus, which was born out of my PhD dissertation research.
In the coming months, we plan to work on some fascinating topics:
 * Running Theseus atop seL4 and other secure hypervisors
 * Executing WASM + WASI binaries on Theseus
 * Implementing support for filesystems, async/await, libc, and libstd

Thanks for reading, and be on the lookout for more content soon. Feel free to contact us via email or on GitHub with comments or questions.

To learn more, use the links up top to explore the Theseus Book and source code. 
