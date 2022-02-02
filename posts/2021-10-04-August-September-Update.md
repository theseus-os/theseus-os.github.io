---
layout: post
title: "August/September Update: A Proper Terminal Emulator"
author: Kevin Boos <https://github.com/kevinaboos>
release: false
---


As mentioned in our [previous update](./2021-08-02-June-July-2021-Update.md), we're working to support headless operation in Theseus such that it can run in an seL4 guest VM, which doesn't support graphical displays.

## Going Headless with a Proper Terminal Interface

We took a three-step approach towards realizing a proper headless interactive terminal:
1. Removing the requirement that a graphical display (e.g., VGA device) must exist for Theseus to successfully boot. 
   * Now, if no display devices are not found, bootstrap and init can proceed, and headless operation is assumed.
2. Devising a new input dataflow and event manager, with the serial port as the focal data source and sink.
   * Monitors input sources for new connections asynchronously and spawns handlers for them upon receival.
3. Create a new terminal emulator that:
   * Offers abstraction layers to support both graphical *and* non-graphical terminals.
   * Supports conventional escape and control codes, compliant with most ANSI, VT100, and xterm.


### Yet Another Serial Port Driver Redesign
Our existing serial port driver was very simple -- it synchronously transmitted output bytes and simply dumped input bytes.
Thus, the receive interrupt handler was trivially simple and executed quickly.

However, the added complexity of doing actual work (e.g., spawning a new console/terminal instance) in the receive interrupt handler caused it to become prohibitively expensive to run.
Executing long-running operations in an interrupt handler is unacceptable: *(i)* it prevents that CPU from doing any other work, and *(ii)* it harms system interactiveness by preventing other interrupts from being handled.

To avoid this, we used Theseus's [asynchronous channels](https://www.theseus-os.com/Theseus/doc/async_channel/index.html) to establish a communication channel between the serial port interrupt handler and a separate *listener* task.
The listener task blocks until receiving a notification from any serial port, upon which it spawns a new console and terminal instance using that serial port and an input source and an output sink.

This design has one drawback: it requires [proactively spawning a dedicated task just to listen for notifications from the serial port](https://github.com/theseus-os/Theseus/blob/3e805f1799964f5b63c48d7ad6a072f130256445/kernel/console/src/lib.rs#L32-L39).
That led us to realize that it was time to introduce a better abstraction: deferred and/or linked interrupt handlers.


#### Deferred Interrupt Handlers

Theseus's [Deferred Interrupt Handlers](https://www.theseus-os.com/Theseus/doc/deferred_interrupt_tasks/index.html) are an extension of the concept of "bottom half" and "top half" interrupt handlers[^1]:
* *Interrupt Handler* (aka "top half"): the short, latency-sensitive function that runs synchronously, immediately when the interrupt request is serviced. It typically does two things:
   1. Notifies the deferred task that work is ready to be done (optionally providing details about that work), and
   2. Acknowledges the interrupt such that the hardware knows it was handled.
* *Deferred task* (aka "bottom half"): the more complex function that runs in a deferred manner to handle longer operations.
   * Runs asynchronously in a non-interrupt context and can thus perform more long-running operations without blocking the rest of that CPU's workloads.


Theseus's deferred interrupt handling implementation is conceptually similar to but differs from tasklets and workqueues in Linux. 
The deferred task is uniquely tied to an interrupt handler in a 1-to-1 manner upon creation and are fully type-safe; there is a direct, strongly-typed channel of communication between them[^2].
The deferred task can also optionally be configured to execute immediately after the interrupt handler in order to reduce I/O device latency, thanks to scheduler integration.

Our favorite side benefit of this design is that itencourages adherence to the *separation of concerns* principle, as the interrupt handling functionality must be conciously divided into an "urgent" synchronous part and a deferred asynchronous part.

With this new interrupt handling architecture, we were able to adapt the serial driver to work efficiently without having unnecessary listener tasks wasting memory and CPU cycles.
The serial port is a simple case, as the interrupt handler only need to notify the deferred task that data has been received on the port and then acknowledge the interrupt; no other data exchange is needed.
The deferred task can then read all received data from the serial port in a non-urgent manner and do anything else necessary, such as spawning new console/terminal instances.


#### Splitting the Serial Port Driver

Theseus's design specifies a *tiny* minimal kernel boot image, the `nano_core` which must include only the bare minimum components to be able to bootstrap the OS and then dynamically load all other OS initialization components. 
One of these components is the serial port driver, as it's the only real usable choice for early boot/kernel logging in the absence of display, networking, or file support.

With the above additions to the serial port driver, namely the usage of deferred interrupts and channels, the driver was becoming far too complex.
It had gone from having zero dependencies to having dozens. 
That bloated the size of the `nano_core`, thereby harming the dynamicness of Theseus.

To solve this, we [split the serial port driver into two parts](https://github.com/theseus-os/Theseus/commit/d6b86b6c46004513735079bed47ae21fc5d4b29d):
1. [Basic driver](https://www.theseus-os.com/Theseus/doc/serial_port_basic/index.html): a standalone crate that exposes the `SerialPort` type, which only offers functions to initialize, read from, and write to the device.
2. [Full driver](https://www.theseus-os.com/Theseus/doc/serial_port/index.html): offers full I/O support with trait implementations, deferred interrupts, channel usage, task blocking, and more. 
   * Wraps the basic driver's `SerialPort` type with additional functionality.


This design enables two previously-conflicting goals:
* The majority of system components can access the full functionality of the serial port and use it just like any other I/O device.
* The kernel boot image is kept tiny with minimal dependencies whilst still being able to output logs to the serial port.


### A Complete Terminal Emulator Rewrite

Finally, we began the most complicated component of true interactive headless operation: a terminal emulator.
Theseus's existing terminal is fairly ad-hoc and doesn't conform to any real standards; it was implemented quickly as a summer intern project and offers just enough features to run commands and display their output.
The same is true for Theseus's `stdio`, which is a "quick-n-dirty" inflexible implementation that uses heap buffers instead of a real file abstraction.

As anyone who has dealt with this knows, terminal emulators are *exceedingly* and unexpectedly complex. 
We experimented with multiple iterations of the design, primarily centered around how best to represent each displayable "unit" or "cell" on the terminal display:

 1. The terminal grid stores a dynamic number of elements per line, in which each element can contain either a single character or a string of characters (no matter how wide) as long as they have the same style.
    * Easy to output to a text backend.
    * The most memory efficient.
    * Calculating line wrapping and cursor navigation is overly complex because each unit may have a different columnar display width.
 2. The terminal grid stores a dynamic number of elements per line, but each element in the line can only contain a single *character*.
    * Slightly less memory efficient, as it duplicates units that have the same style.
    * Easier to calculate displayed rows and cursor movement than design 1, but still complex.
 3. The terminal grid stores a dynamic number of elements per line, but each element corresponds to a *single displayed column* on screen.
    * Requires more memory and a little bit of extra effort to support wide characters, since they must be split across more than one element.
    * Much, much easier to calculate cursor movements, line wrapping, and display commands.
 4. The terminal grid stores a fixed number of elements per line, equivalent to the width of the screen row; blank units are inserted to fill each linas necessary.
    * Very wasteful of memory, especially for short lines (common in terminal output).
    * Trivial to calculate displayed rows and cursor movement.
    * Very hard to re-size/re-flow the terminal.

After multiple iterations, we settled on **Design 3** above.
It selects the best set of tradeoffs in the spectrum of design points, and most importantly, simplifies the translation between on-screen coordinates and scrollback buffer coordinate.

One interesting aspect of Theseus's terminal emulator is how it's split into multiple components:
 * The *frontend*: a single entity responsible for handling incoming character and control/escape code bytes and determining what actions should be taken.
   * The frontend is also reusable as a terminal driver, `readline` library, and line discipline layer.
 * The *backend* abstraction: a trait that represents the various display actions that a terminal might invoke.
 * The *backend* implentation(s): one of multiple entity types that handles display actions to "render" the terminal output in different formats:
   * Graphical pixel framebuffers
   * Classic 80x25 VGA screen with 16-color basic text display
   * A serial port connected to a host-side pseudo-terminal
   * A network connection or file or any other sink


However, this is still a work in progress and is not yet used for Theseus's main terminal display.
It takes quite a lot of effort to get all the details right, but we have an MVP that supports:
* Insert/replace mode switching
* Automatic CR/LF behavior with wrapping
* Multiple different PTY backends, e.g., `screen`, `picocom`, `minicom`, etc
* Most ANSI escape/control codes, including colors and text styles
* Cursor movement and scrolling

Fortunately, there are several Rust terminal emulators that serve as valuable examples, e.g., [Alacritty](https://github.com/alacritty), but they're standalone monolithic projects that rely on an underlying graphics stack like OpenGL and are thus inappropriate for in-kernel usage.
We were able to use the excellent [`vte`](https://crates.io/crates/vte) crate from the Alacritty project though, which saved a lot of time and effort in parsing ANSI/VT100 control codes.

> Did you know that the "grid" of a terminal screen is indexed starting at `(1,1)` rather than `(0,0)` for the upper-leftmost character? <sup> Oh, how I wish I knew that earlier...</sup>



## Other Improvements
* Improved safety, complexity, and performance of [interpreting a memory region as an executable function](https://github.com/theseus-os/Theseus/pull/419).
   * Previously called `MappedPages::as_func()`, now moved into the crate management subsystem as a member function of the `LoadedSection` type.
   * The new [`LoadedSection::as_func()` method](https://www.theseus-os.com/Theseus/doc/crate_metadata/struct.LoadedSection.html#method.as_func) already knows both the size of the function's section and whether the underlying memory is executable, which omits multiple runtime checks.
   * The function interface is much simpler, as the caller need not specify any size or offset values w.r.t. the memory region.
   * Improves safety: offers a compile-time guarantee that an executable function instance can only be obtained from a `LoadedSection` (with proper alignment and size) rather than any chunk of executable memory.

* Re-designed parts of the [`Task` struct to reduce lock contention](https://github.com/theseus-os/Theseus/commit/df721e35221d361f8ec8fd87364133f0be0f5cde), eliminating it in most places.
   * "Mutable" fields (those that may change after `Task` creation) are now moved into a `TaskInner` struct (excluding atomic fields), which allows most fields to be accessed without acquiring the lock on that `Task`.
   * Immutable fields are kept in the "outer" `Task` struct. 
   * We ensure atomic types are properly aligned and sized such that native atomic instructions are actually used. We realized this by switching from `atomic::Atomic` to `crossbeam_utils::atomic::AtomicCell`, plus static assertions that all types wrapped in `AtomicCell` are actually eligible for native atomic access.
   * Move a Task's `ExitValue` out of its `RunState` enum so that the `RunState` enum can be accessed atomically and independently of an `ExitValue`.
   * The `runstate` and `running_on_cpu` fields of `Task` are now atomically accessible in a lock-free manner, allowing us to block/unblock tasks in lock-free contexts, e.g., within an interrupt handler.
   * As more `Task` fields are now readable without locking, we can remove all now-unnecessary unsafe statements from the `scheduler`.

[^1]: Other terminology is often used, including "first-level" and "second-level" interrupt handlers, or "hard" and "soft" interrupt handlers.
[^2]: It is also possible to have a pool of deferred tasks, in which each task can be tied to multiple interrupt handlers.
