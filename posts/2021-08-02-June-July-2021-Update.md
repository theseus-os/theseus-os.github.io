---
layout: post
title: "June/July Update: Headless Operation on seL4"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---

## One Very Busy, Very Hot Summer 

In June of this year, the Seattle area hit record high temperatures of over 110°F (44°C) for three days. Ouch!

We began this hot summer with the goal of enabling Theseus to run atop seL4, using both its hypervisor and VMM functionality to present Theseus with a standard "bare metal" x86 environment.
Unfortunately, we quickly discovered that while [seL4 supports ARM and x86](https://docs.sel4.systems/Tutorials/camkes-vm-linux.html), it does not yet fully support x86_64 VMMs, which is the only architecture that Theseus currently runs on.
The implementation of an x86_64 VMM was supposedly [completed by Dornerworks](https://dornerworks.com/blog/64-bit-x86-architecture-on-sel4/), but we were unable to get it to successfully run any x86_64 guest OS (but x86 VMs did work properly).
We have decided to postpone this particular effort until [this PR](https://github.com/seL4/seL4/pull/324) that officially adds support for x86_64 VMMs on seL4 is accepted.

## Headless operation 
In the meantime, we started working towards Theseus-level support for _headless_ operation over a serial port interface.
The serial port is the only form of direct interactive access to guest VMs on seL4 (excluding network access), so it is a necessary component to debug and use Theseus therein.
It's also useful for communicating with serial devices on other more limited platforms, e.g., our [WIP port of Theseus to ARM Cortex-M4 microcontrollers](https://github.com/theseus-os/Theseus/pull/361).

Previously, Theseus had two problems in this area:
 1. All input to the shell/terminal was assumed to come from a real keyboard and mouse.
 2. The serial port was only used for basic logging output (not treated as a regular I/O device).

To achieve headless operation, we had to set two corresponding goals:
 1. Abstract the terminal and input handling to work with any I/O source, not just a physical keyboard.
 2. Enable full, bidirectional, arbitrary I/O across serial ports.


### New I/O Abstractions:  Stateless vs. Stateful
As the first step towards these goals, we created [several new I/O traits](https://github.com/theseus-os/Theseus/blob/051e52782658a3e0f11c486d8656e71da1f7ba07/kernel/io/src/lib.rs) to represent different categories of I/O. 

 * The [`BlockReader`] and [`BlockWriter`] traits represent I/O streams which can be read from or written to at the granularity of a single block (as the smallest transferable chunk).
 * [`BlockIo`] is a "parent" trait that specifies the size in bytes of each block 
   in a block-based I/O stream.
 * [`KnownLength`]: a separate trait that represents an I/O stream with a known length, such as a disk drive.
 * [`ByteReader`], [`ByteWriter`]: traits that represent I/O streams which can be read from or written to at the granularity of an individual byte.
 * We also provide wrapper types that allow byte-wise access atop block-based I/O streams: [`ByteReaderWrapper`], [`ByteWriterWrapper`], [`ByteReaderWriterWrapper`].
    * The [`blocks_from_bytes()`] function is useful for calculating the set of block-based I/O transfers that are needed to satisfy an arbitrary byte-wise transfer.

Notably, these traits all offer ***stateless*** access to byte streams only, an important behavioral characteristic that helps simplify state management in Theseus.
This means that they don't keep track of an internal offset within the stream.

For example, the `ByteReader` trait exposes only one function that requires the caller to specify at which offset the stream read should start.
```rust
fn read_at(&mut self, buffer: &mut [u8], offset: usize) -> Result<...>
```

These traits and types also *stack* on top of each other, e.g., you can use a `ByteReader` to realize byte-wise access to an underlying block-based I/O device that implements `BlockReader`. 
We make this easier with trait *delegation*, in which wrapper types "forward" the trait implementation through:
 * References (`&dyn ByteReader`)
 * Mutable References (`&mut dyn ByteReader`)
 * Locks (`Mutex<ByteReader>`) using the [`LockableIo`] type


We also offer ***stateful*** I/O types, which wrap stateless I/O streams (the above traits) to track the current offset into the I/O stream while reading or writing it. 
This is similar to classic POSIX I/O interfaces, but are strongly-typed and allow for limited permissions: [`ReaderWriter`], [`Reader`], and [`Writer`] structs.

Finally, all of the above types and traits implement the `no_std` version of `std::io::Read`/`Write` traits, which can come from crates like [`core_io`](https://crates.io/crates/core_io) or [`bare_io`](https://crates.io/crates/bare-io).
This widely expands their compatibility to work with pretty much any other I/O-related code in the Rust ecosystem.

For example, we used these new I/O abstractions to integrate the [`fatfs` Rust crate](https://github.com/rafalh/rust-fatfs) with Theseus, which allows Theseus to read and write the contents of a FAT filesystem on disk.
More work is required to provide a more generic file abstraction that can represent arbitrary files across any filesystem type, as Theseus's current representation of files is quite ad-hoc and limited to in-memory filesystems.


### Redesigned Serial Port Driver
With redesigned I/O traits, we can proceed to our second goal: improving the serial port driver.

On x86 machines, there are up to 4 serial ports, but commonly only one or two are available: `COM1` and `COM2`. 
The OS can interact with them using different I/O ports, e.g., writing bytes to `0x3F8` and the subsequent 7 port addresses will allow you to communicate with the `COM1` serial port.
Here are three great resources for learning more about serial port behavior and how to write a proper driver: [one](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming), [two](https://tldp.org/HOWTO/Modem-HOWTO-4.html), [three](https://wiki.osdev.org/Serial_Ports).


The changes we needed to make are:
* Implement and activate interrupt handlers for all serial ports, such that the hardware triggers an interrupt when input bytes are received on the port.
* `SerialPort` instances are no longer exclusively owned by the logger, as they must be accessible from within the serial port interrupt handler and other kernel/application crates.
* Implement the necessary read and write I/O traits for the `SerialPort` type, so we can use them in all I/O stream contexts.

With these changes in place, Theseus is now able to read from and write to serial ports freely, as if it were any other I/O stream like a file or disk.


## Other Improvements to Theseus
* Updated Theseus's Rust compiler to version 1.54, which entailed [many changes](https://github.com/theseus-os/Theseus/commit/b7d62ee0197347b651e2cf1387f83c9c4a598633):
   * Refactoring all inline assembly to Rust's new `asm!()` syntax.
   * Complying with the restrictions on naked functions: Rust ABI is no longer allowed, and only one assembly block is permitted per naked function.
   * Use the new [`compare_exchange_weak()`](https://doc.rust-lang.org/stable/core/sync/atomic/struct.AtomicUsize.html#method.compare_exchange_weak) family of functions, which is more efficient on some architectures (ARM) because it is allowed to spuriously fail.

* Refactored code for memory-related types to unify their APIs:
   * [`VirtualAddress` and `PhysicalAddress`](https://github.com/theseus-os/Theseus/pull/417)
   * [`Page` and `Frame`](https://github.com/theseus-os/Theseus/commit/6854306a8f2c16f3caf1332120856a0fff8de25f)
   * [`PageRange` and `FrameRange`](https://github.com/theseus-os/Theseus/commit/b09ad9bc73683397a0b16b9b53f9214bdf87c04d)

* Book documentation: thanks to [@apogeeoak](https://github.com/apogeeoak), the Theseus Book now has clearer structure, automatic spell check, and is built and published online via GitHub Actions CI workflows. 

* Mellanox 100GiB NIC: [Ramla Ijaz](https://github.com/Ramla-I) added basic support for [initializing and configuring this high-performance NIC](https://github.com/theseus-os/Theseus/pull/404). Packet transmission is an ongoing work in progress.

* Added support for parsing the ACPI `DMAR` table, which specifies details about the system's IOMMU. 
   * This is part of our quest to protect Theseus's single address space execution environment from errant or malicious I/O devices that attempt to access arbitrary system memory without permission, the one final frontier in the "chain of safety" that cannot be checking by the compiler.


## Contributions to other Open-Source Projects
* We added [compile-time configuration of logging flexibility](https://github.com/rafalh/rust-fatfs/pull/44) to the `rust-fatfs` crate.
* We ported the MPMC Queue crate to [support `no_std` environments](https://github.com/brayniac/mpmc/pull/8) on the latest version of Rust.


## Next Steps
Now that we have flexible, generic I/O abstractions available in Theseus, the next step for achieving full headless operation is to enable a terminal/CLI to handle I/O to and from arbitrary sources, such as a serial port.


[`BlockReader`]: https://www.theseus-os.com/Theseus/doc/io/trait.BlockReader.html
[`BlockWriter`]: https://www.theseus-os.com/Theseus/doc/io/trait.BlockWriter.html
[`BlockIo`]: https://www.theseus-os.com/Theseus/doc/io/trait.BlockIo.html
[`KnownLength`]: https://www.theseus-os.com/Theseus/doc/io/trait.KnownLength.html
[`ByteReader`]: https://www.theseus-os.com/Theseus/doc/io/trait.ByteReader.html
[`ByteWriter`]: https://www.theseus-os.com/Theseus/doc/io/trait.ByteWriter.html
[`ByteReaderWrapper`]: https://www.theseus-os.com/Theseus/doc/io/struct.ByteReaderWrapper.html
[`ByteWriterWrapper`]: https://www.theseus-os.com/Theseus/doc/io/struct.ByteWriterWrapper.html
[`ByteReaderWriterWrapper`]: https://www.theseus-os.com/Theseus/doc/io/struct.ByteReaderWriterWrapper.html
[`blocks_from_bytes()`]: https://www.theseus-os.com/Theseus/doc/io/fn.blocks_from_bytes.html
[`LockableIo`]: https://www.theseus-os.com/Theseus/doc/io/struct.LockableIo.html
[`ReaderWriter`]: https://www.theseus-os.com/Theseus/doc/io/struct.ReaderWriter.html
[`Reader`]: https://www.theseus-os.com/Theseus/doc/io/struct.Reader.html
[`Writer`]: https://www.theseus-os.com/Theseus/doc/io/struct.Writer.html
