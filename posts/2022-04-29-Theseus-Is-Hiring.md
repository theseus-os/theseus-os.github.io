---
layout: post
title: "Theseus is Hiring!"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---


## Theseus is looking for systems programmers!

Theseus is proud to be a member of a diverse ecosystem of open-source Rust projects sponsored by Futurewei Technologies.
Thanks to the generosity of Futurewei's Rust initiative, Theseus is able to hire developers for the first time!

**To apply, send your resume or C.V. to [theseus.systems@gmail.com](mailto:theseus.systems@gmail.com).**


Check out the below sections for more info.

![Theseus plus Futurewei](/images/2022-posts/theseus_plus_futurewei.svg)

### Job Title: Operating Systems Developer

As an OS developer, you will independently lead a major development project within Theseus OS.

To lend some context, Theseus OS is a novel operating system written from scratch entirely in Rust,
with the objective of realizing next-generation safety and efficiency guarantees for workloads in a variety of execution environments. 
Theseus's goals span the gamut from supporting cutting-edge exploratory research topics to more practical concerns of achieving legacy compatibility and usability.

Lately, our focus is on deep support for WebAssembly (WASM) interfaces and runtimes, as well as porting Theseus to additional architectures (e.g., ARM).
As such, your development project ([more ideas below](#project-ideas)) will likely be related to one of those areas.

Other job responsibilites include:
* Collaborating with the general public in an open-source environment
* Addressing issues and solving bugs/problems expediently
* Writing easily readable code and documenting it well
* Meeting with other Theseus OS developers, Rust developers, and Futurewei team members 


### Location and more

* Fully remote
* Flexible work hours
  * Ideally 2+ hours of daily overlap with the US Pacific time zone
* Work-life balance is highly valued and respected
* Required equipment: any standard PC capable of running Linux/Windows


### Compensation and Term

$45-$75 USD per hour (approx. $7500-$12,000 USD monthly), commensurate with experience.

The initial employment term is:
* 3-4 months if full-time (~40 hours per week) 
* 6-9 months if part-time (~20 hours per week)

Terms can be extended if the candidate performs well. 


### Eligibility

Theseus is a U.S.-based organization but does *not* require citizenship, permanent residency, or work visa/authorization.

Anyone from any country is welcome to apply, including but not limited to students, professionals, open-source developers, hobbyists, and more.
If in the U.S., you will be hired as a standard 1099 independent contractor, as with typical freelance work.


### Desired Skills

The ideal candidate would have experience with at least one of the following:

* Systems programming languages
    * Rust preferred
    * C, C++, or Assembly appreciated
* Operating Systems concepts
    * Multithreading and concurrency
    * Memory management, virtual memory
    * Architectural knowledge, e.g., ARM/x86
* Basic compiler or programming language concepts
    * Type safety, memory safety
* WebAssembly, sandboxing, software isolation
* Source code versioning via `git` and the general GitHub workflow
* Working independently in a self-driven context
* Clear communication and technical writing skills (for documentation)


If you're not an expert in these areas, don't worry! You'll pick up any necessary skills while working with us. As academics at heart, we love teaching these concepts, and it's always exciting to watch new OS concepts click.


### What You'll Learn

* Deep knowledge of how operating systems work under the hood
* Expertise in the Rust programming language
* How to write **_safe_**, easy-to-read, and robust code
* Experience with navigating/understanding large projects
* Skills for producing good documentation
* Best practices for contributing to public open-source projects


### Project Ideas

Projects are fairly open-ended and flexible; the topic is ultimately up to you, but should ideally relate to one of Theseus's current development goals. We do encourage you to work on something that interests you personally.
> If you've ever caught yourself thinking _“I've always wanted to know how XYZ works in an OS”_ then this is where you'll shine (and get paid to follow your curiosity!).

Your project(s) will expand or improve upon the existing functionality of Theseus or add new features entirely. We have an extensive list of exciting project ideas that you can tackle or use as inspiration for an idea of your own: 

* Extending support for running WebAssembly (WASM) on Theseus
  * Supporting more WASM interfaces, e.g., WASI extensions
  * Improving WASM performance (e.g., with `wasmtime`) atop Theseus
* Creating a generic universal driver abstraction for cross-OS driver reuse
  * Ask us about `WASI-dd`: a future WASM-based interface for reusing drivers on any WASM-compliant OS
* Deeper graphics support and a better graphics stack
  * Supporting WASM+WebGL workloads
  * Creating a new more featureful window manager or compositor
* Legacy compatibility: improving Theseus's implementation of Rust's `std` library and/or `libc`
* Porting Theseus to another architecture, e.g., ARM `aarch64`, RISC-V
* Driver development for additional devices and peripherals
  * Storage devices, GPUs, USB devices, audio chips, etc.
  * Proper power management, ACPI shutdown, device/CPU suspend & resume
* Language-level: deeper integration and support for Rust's async/await syntax
* See [more project ideas here](https://github.com/theseus-os/Theseus/wiki) 


### About Us: our Vision and Culture

We strongly believe that Theseus OS is well-positioned to be the next great system for supporting complex dynamic workloads on both low-end and high-end embedded systems, datacenter servers, and any other single-operator computing environment **where safety and efficient performance are key**.

> * Theseus began as a research-oriented OS that focused on investigating the benefits of a unique OS structure and novel methods of state/resource management.
> * Theseus has received positive feedback from experts in academia, appearing in top-tier conferences like the Symposium on Operating Systems and Principles (SOSP) in 2020.

We are in the process of transitioning Theseus from a research prototype to a more usable fully-fledged operating system that will support arbitrary workloads, both existing popular applications and libraries as well as up-and-coming interfaces, e.g., WASM and its interface extensions.

> However, we still retain that academic culture and feel of fearlessly exploring wild and wacky ideas without concern for commercial usability.

As with most open-source projects, the work environment is casual and chill. While we do expect you to make progress, we are also all too familiar with the challenges inherent in OS development and are very understanding of unforeseen delays or complications. We'll be there to guide you to overcome any difficulties/setbacks along the way.

You'll work directly with Kevin Boos, the creator of Theseus OS and founder of Theseus Systems. Kevin is the full-time lead developer and will work personally with you to specify your project, implement your design, and foster knowledge of the fundamentals of Rust, Theseus, and OS functionality.

You'll also interact with a variety of other contributors, such as technical managers and experts from Futurewei or systems professors and PhD students from Yale University and other academic institutions.

Theseus OS is a fully open-source project that welcomes contributions from anyone. Thus, you may interact with the Rust or OS community at large, both of which are often recognized as the most welcoming of all online developer communities. Rustaceans are super friendly!

We recognize your time is valuable: meetings are kept to a minimum! That being said, Kevin and others are always happy to meet with you and help you out at any time of day. Project guidance can be as hands-on or hands-off as you like, according to your preferred work style.


|                     |                   | 
| :-----------------: | :---------------: |
| ![Photo of Kevin Boos](/images/2022-posts/kevin-boos.png "Kevin Boos") | ![Photo of Puma](/images/2022-posts/puma.png "Puma the Pup") | 
| <div align="center"> **Kevin Boos** <br> Founder & Creator of Theseus <br> PhD, Rice University </div> | <div align="center"> **Puma** <br> Chief Morale Officer <br> PhD in Ball Fetching, sneaky Zoom ninja </div> |


Thanks, and we look forward to meeting you!