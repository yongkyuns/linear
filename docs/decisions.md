# Linear Design Decisions

This document is the review-oriented record of architectural decisions for Linear.

The detailed architecture remains in [`architecture.md`](architecture.md). This file answers a different question: **what have we decided, why, and what is still open?**

## Status legend

- **Accepted** — current design direction; implementation should preserve it unless explicitly revisited.
- **Provisional** — favored direction, but important details remain open.
- **Open** — not yet decided.
- **Rejected** — considered and intentionally not chosen.

## Decision index

| ID | Topic | Status | Decision |
|---|---|---|---|
| D001 | Application concurrency model | Accepted | Kernel provides mechanisms; threads, Active Objects, state machines, and Rust async are optional application/framework models. |
| D002 | Active Objects | Accepted | AO is a library pattern built from ordinary OS primitives, not a kernel primitive. |
| D003 | POSIX role | Accepted | POSIX is an application-facing compatibility profile, not Linear's internal ontology. |
| D004 | Testability | Accepted | Dependency boundaries are test seams; code must be runnable at component, partial-graph, whole-userspace, and system levels. |
| D005 | OS time under test | Accepted | Kernel time must support deterministic virtual/accelerated time for timers, sleeps, timed waits, and simulated devices. |
| D006 | General memory-allocation philosophy | Accepted | Dynamic allocation is enabled by default; allocation policy is configurable by product/profile rather than baked into application APIs. |
| D007 | Kernel allocation policy | Accepted | General profile follows the flexible Zephyr/NuttX camp; stronger deterministic/safety profiles may restrict or eliminate runtime heap allocation without redesigning APIs. |
| D008 | Memory protection / execution domains | Open | Next decision. |

---

## D001 — Concurrency policy stays above the kernel

**Status:** Accepted

### Decision

Linear provides general-purpose mechanisms such as:

```text
threads
mutexes / condition variables
semaphores
bounded queues / channels
events / notifications
timers
wait sets / poll-like waiting
priority-aware scheduling
ISR-safe signaling
```

It does **not** make one application concurrency model fundamental to the RTOS.

Applications may choose ordinary blocking threads, Active Objects/state machines, an optional Rust async executor such as Embassy, or another model.

### Rationale

Trying to make blocking and Rust async interfaces look identical introduces significant complexity and async coloring. Runtime injection or stackful-fiber mechanisms can also become architectural machinery that conflicts with Linear's goal of supporting simple ordinary `std` code.

The kernel should therefore provide mechanisms, while concurrency policy remains above it.

---

## D002 — Active Objects are a library pattern

**Status:** Accepted

### Decision

Active Objects are supported cleanly by building them from existing OS facilities such as threads, channels/message queues, timers, events, and priorities.

AO is not a kernel object and is not required by the service model.

A user may implement:

```text
1 AO = 1 RTOS thread
many AOs = 1 run-to-completion worker
many AOs = N-worker pool
```

without changing the kernel ABI.

### Rationale

AO localizes asynchronous behavior at message boundaries while keeping event handlers and domain logic ordinary synchronous code. Keeping AO above the kernel preserves runtime neutrality and makes the same component logic easy to test without spawning its production runtime.

---

## D003 — POSIX is a compatibility profile

**Status:** Accepted

### Decision

Linear aims for a useful embedded POSIX profile and practical Rust `std` support, but does not model its internal kernel architecture around Unix abstractions.

Native typed service/device interfaces may coexist with POSIX file descriptors, `/dev`, sockets, pthreads, and libc.

### Rationale

POSIX is valuable for portable C/C++ software and makes Rust `std` platform integration more practical. Full Unix process/session/signal semantics should not distort MCU-oriented architecture when they provide little value.

---

## D004 — Testability through substitution is a core architecture principle

**Status:** Accepted

### Decision

> Every externally observable dependency boundary should also be a test seam where practical.

Production dependencies must be replaceable by compatible test implementations without modifying the code under test.

Examples:

```text
Production                 Test
STM32 SPI                  Fake/Mock SPI
BMI270                     Simulated BMI270
u-blox GNSS                Replay/simulated GNSS
network                    Fake network
flash                      Memory/host storage
IRQ/DMA                    Injected completion/events
system services            Fake/replay providers
```

Linear should support testing at multiple granularities:

```text
L0 pure algorithm/data structure
L1 component/service logic
L2 driver logic with fake HAL/device
L3 partial service graph
L4 complete userspace graph on host
L5 kernel mechanisms with deterministic host tests
L6 kernel + userspace in simulated/emulated environment
L7 hardware integration
L8 full product/system
```

### Rationale

The same abstractions used for portability and composition should also enable modular testing. Users must be able to run code in pieces or as a complete graph, selecting the smallest useful scope for each test in the test pyramid.

Behavioral fakes and device models are preferred over brittle call-count mocks when realistic semantics are more useful. Fault injection is a first-class capability of test providers.

---

## D005 — OS time is virtualizable and controllable under test

**Status:** Accepted

### Decision

Kernel timekeeping must not be intrinsically tied to the physical timer peripheral.

The timer subsystem consumes an abstract monotonic time source:

```text
                   kernel time/timer subsystem
                            |
                    monotonic source
                    /              \
             hardware time       virtual time
```

The virtual test environment should be capable of controlling all OS facilities derived from monotonic time, including:

```text
std::time::Instant
std::thread::sleep
clock_gettime(CLOCK_MONOTONIC)
nanosleep
pthread timed waits
semaphore/channel timeouts
poll/select/socket timeouts
kernel timers
AO timers
simulated IRQ/DMA/device events
```

Desired test modes:

```text
real time
scaled time (e.g. 10x, 100x)
manual deterministic time
next-event / auto-advance discrete-event time
```

`CLOCK_REALTIME`/calendar time must remain logically separate from monotonic time so wall-clock adjustments do not invalidate deadlines.

### Rationale

Normal application code should not need explicit `Clock` injection merely to make `std` sleeps and OS timeouts testable. Virtualizing the OS time source allows complete userspace/system tests to execute long-duration behavior rapidly and deterministically, similar in goal to `libfaketime` but built into the OS architecture rather than implemented as interception.

Virtual time also enables deterministic scheduler tests and a common timeline for timers, simulated interrupts, DMA completions, and device models.

### Open details

- periodic timer semantics when virtual time jumps over multiple periods;
- catch-up vs skip/coalesce policy;
- ordering of simultaneous timer, interrupt, and device events;
- exact host test-controller API (`advance`, `run_until_idle`, `run_next_event`, etc.).

---

## D006 — Dynamic allocation is enabled by default

**Status:** Accepted

### Decision

The default Linear configuration provides ordinary dynamic allocation for userspace so portable Rust `std`, C, C++, networking, filesystems, and third-party libraries remain ergonomic.

Examples that should work naturally in the general profile:

```rust
Vec::new();
Box::new(value);
String::new();
std::thread::spawn(...);
```

Heap availability is a **product/profile configuration choice**, not something service and application interfaces should encode into their type signatures.

Linear should avoid spreading allocator generics through ordinary APIs solely to support stricter builds later.

Special memory semantics remain explicit, for example:

```text
DMA-safe memory
non-cacheable/coherent memory
TCM
retention RAM
fixed packet pools
special realtime pools
```

### Rationale

Making `std` a first-class application goal conflicts with treating dynamic allocation as universally forbidden. The architecture should preserve normal language ergonomics while allowing products with stronger determinism requirements to tighten policy through configuration.

---

## D007 — Kernel allocation policy is profile-driven, not universally static

**Status:** Accepted

### Decision

Linear will **not** impose the safety-RTOS rule that the kernel can never use a general-purpose heap at runtime.

The default/general profile follows the more flexible Zephyr/NuttX design family:

```text
Linear General
  userspace heap             enabled by default
  kernel dynamic allocation  permitted when configured
  runtime object creation    supported
  fixed pools/slabs          available
  static allocation          available
```

Linear must nevertheless be designed so stronger profiles can tighten the same system without changing application/service APIs:

```text
Linear RT / Deterministic
  general allocation allowed in noncritical areas
  bounded pools/slabs usable for critical paths
  critical-path allocation policy auditable/configurable

Linear Safety
  kernel runtime heap may be disabled/restricted
  kernel objects may be statically provisioned or drawn from bounded pools
  userspace allocation may be restricted or disabled after initialization
  resource maxima may be explicitly declared and validated
```

The important invariant is therefore **configurability of allocation policy**, not a universal allocation rule.

### Rationale and comparison

Current RTOSes occupy different positions:

- **NuttX** strongly favors Unix/POSIX flexibility and makes substantial use of dynamic memory, including separate kernel/user heaps in protected configurations and specialized allocators where needed.
- **Zephyr** provides configurable general heaps together with fixed-size slabs and bounded allocation options, allowing the product to choose stronger determinism where required.
- **FreeRTOS** supports both dynamic and static kernel-object creation, including configurations where dynamic allocation is disabled.
- **SAFERTOS** represents the stricter safety-oriented end: RTOS memory is supplied during initialization and the RTOS itself avoids dynamic allocation.

Linear intentionally starts closer to the **Zephyr/NuttX flexibility camp** while preserving a path toward the deterministic/safety end of this spectrum. This is important for broad `std` compatibility and developer ergonomics while still allowing future use in reliability-sensitive domains such as medical devices.

### API principle

Allocation policy should normally remain below application APIs. Avoid forcing users to select parallel APIs such as `create()` versus `create_static()` unless different semantics genuinely require it.

Ideally:

```rust
Thread::spawn(...)
```

can be backed by different configured allocation strategies across product profiles.

### Observability requirement

Even the General profile should expose memory diagnostics sufficient to evaluate reliability and migrate a product toward stricter profiles:

```text
heap current / peak usage
allocation failures
fragmentation metrics where meaningful
pool utilization
per-domain usage
thread stack high-water marks
special-memory usage
```

The composition/build system should eventually report statically known memory budgets where possible.

### Still open within memory design

The following are not yet frozen and can be decided when concrete allocators/kernel objects are designed:

- default kernel allocator implementation;
- whether individual kernel subsystems use dedicated pools by default;
- stack allocation classes/arenas;
- memory-domain accounting;
- post-init allocation controls for deterministic/safety profiles;
- exact safety-profile guarantees and certification-oriented restrictions.

---

## Next decision

### D008 — Memory protection and execution domains

**Status:** Open

To decide whether Linear should architecturally support flat, MPU-protected, and MMU/process execution domains from the beginning, and whether the kernel/userspace boundary should compile down to direct calls in flat configurations.

No decision has been made yet.
