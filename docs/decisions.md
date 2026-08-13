# Linear Design Decisions

This document is the review-oriented architecture decision record for Linear.

The detailed narrative architecture remains in [`architecture.md`](architecture.md). This file answers a different question: **what have we decided, why, and what still needs a decision?**

The IDs in this document are stable. New decisions should get a new ID rather than renumbering existing entries.

## Status legend

- **Accepted** — current design direction; implementation should preserve it unless explicitly revisited.
- **Provisional** — favored direction, but important details remain open.
- **Open** — not yet decided.
- **Deferred** — intentionally postponed until another decision or implementation experience provides more information.
- **Rejected** — considered and intentionally not chosen.

---

# Decision index

## A. Core architecture and application model

| ID | Topic | Status | Decision / question |
|---|---|---|---|
| D001 | Application concurrency model | Accepted | Kernel provides mechanisms; threads, Active Objects, state machines, Rust async, and other runtimes remain optional models above it. |
| D002 | Active Objects | Accepted | AO is a library pattern built from ordinary OS primitives, not a kernel primitive. |
| D003 | POSIX role | Accepted | POSIX is an application-facing compatibility profile, not Linear's internal ontology. |
| D004 | Testability | Accepted | Dependency boundaries are test seams; code must be runnable at component, partial-graph, whole-userspace, and system levels. |
| D005 | OS time under test | Accepted | Kernel time supports deterministic virtual/accelerated time for timers, sleeps, timed waits, and simulated devices. |
| D006 | General memory allocation | Accepted | Dynamic allocation is enabled by default; allocation policy is configurable by product/profile rather than encoded in application APIs. |
| D007 | Kernel allocation policy | Accepted | General profile follows the flexible Zephyr/NuttX camp; deterministic/safety profiles may progressively restrict allocation. |

## B. Memory, protection, and authority

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D008 | Memory protection / execution domains | Open | Should flat, MPU-protected, and MMU/process profiles share one architecture from the beginning, and should protection boundaries compile away in flat builds? |
| D009 | Kernel/user syscall boundary | Open | What is the native syscall model, and how does it map to direct calls in flat builds and validated transitions in protected builds? |
| D010 | Kernel object / handle model | Open | Are kernel resources represented by raw pointers, typed handles, capability handles, integer IDs, or a layered combination? |
| D011 | Capability / authority model | Open | Should possession of a service/object handle also define access authority, and can composition enforce least privilege? |
| D012 | Memory regions and allocators | Open | How are general RAM, DMA-safe memory, TCM, non-cacheable memory, retention RAM, slabs/pools, and per-domain heaps represented? |
| D013 | Stack model | Open | How are stacks provisioned, guarded, measured, and allocated for static and runtime-created threads? |
| D014 | Buffer ownership and zero-copy | Open | How are DMA, IPC, networking, loaned buffers, pinning, cache coherency, and cross-domain ownership modeled safely? |
| D015 | Thread-local storage | Open | What TLS model supports `errno`, Rust/C/C++ TLS, destructors, and protected execution domains? |

## C. Kernel execution and real-time semantics

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D016 | Scheduling policy | Open | Fixed-priority preemptive only, round-robin at equal priority, cooperative mode, deadline classes, or multiple scheduling classes? |
| D017 | Scheduling guarantees | Open | Which kernel paths and wakeup latencies must be bounded, and what deterministic guarantees are part of the architecture contract? |
| D018 | Synchronization semantics | Open | Define fairness, priority ordering, timeout behavior, cancellation, recursion, ownership, and priority inheritance/ceiling for mutexes, condvars, semaphores, events, queues, and wait sets. |
| D019 | Interrupt model | Open | Hard ISR vs deferred work/threaded IRQs, nesting, priority rules, ISR-safe API subset, affinity, and latency guarantees. |
| D020 | Kernel deferred work | Open | Should the kernel have standardized work queues/bottom halves, or leave deferred execution entirely to higher layers? |
| D021 | SMP / multicore | Open | Which single-core assumptions must be avoided now, and what eventual model covers affinity, migration, IPIs, locking, per-CPU state, and cache coherency? |
| D022 | Cancellation and thread termination | Open | What can be cancelled, how is cancellation observed, and are forced thread termination or asynchronous cancellation allowed? |
| D023 | Kernel timing semantics | Open | Finalize timer ordering, periodic timer catch-up/skip/coalesce rules, simultaneous events, and time behavior across suspend. |

## D. Errors, failure, lifecycle, and recovery

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D024 | Error model | Open | How do native errors, `errno`, Rust `Result`, C APIs, driver failures, exhaustion, and programmer errors map across layers? |
| D025 | Panic / crash policy | Open | What happens on Rust panic, kernel invariant violation, userspace crash, or fatal driver failure? Abort, unwind, restart, reset, safe mode? |
| D026 | Service failure and recovery | Open | Can services restart independently, degrade gracefully, or rebind providers after failure? |
| D027 | Boot and initialization lifecycle | Open | Define boot phases, dependency initialization, blocking/allocation rules, deferred/lazy initialization, failure propagation, shutdown, and reboot. |
| D028 | Device lifecycle | Open | Define device states and rules for initialization, open/close, suspend/resume, failure, restart, removal/hot-plug, and reconfiguration. |
| D029 | Watchdog / health supervision | Open | What kernel/application health-monitoring model supports watchdogs, liveness checks, fault containment, and recovery policies? |

## E. HAL, drivers, and hardware composition

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D030 | Driver execution model | Open | In which context may driver code execute: caller, ISR, kernel worker, dedicated thread; who serializes access and where do callbacks run? |
| D031 | HAL semantic contract | Open | What behaviors—not just method signatures—must portable HAL implementations guarantee? |
| D032 | HAL/driver conformance tests | Open | What reusable test suites must every HAL/driver implementation pass? |
| D033 | Board/device description format | Open | Devicetree-compatible, neutral schema, Rust DSL, or generated representation? |
| D034 | Resource resolver | Open | Which conflicts and constraints should be statically validated: pin/IRQ/DMA conflicts, power dependencies, memory, priority rules, capabilities, etc.? |
| D035 | Driver extensibility / escape hatches | Open | How do portable interfaces expose vendor-specific capabilities without destabilizing common APIs? |

## F. Services, IPC, and system composition

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D036 | Service contract representation | Open | Rust traits, C vtables, generated language-neutral IDL, stable handles, or a combination? |
| D037 | Service discovery and binding | Open | Compile-time injection, capability IDs, names, runtime registry, or hybrid? |
| D038 | Optional and multiple providers | Open | How are optional capabilities, fallback providers, ranking, late availability, and controlled degradation represented? |
| D039 | Service graph lifecycle | Open | Is the graph static after boot, partially dynamic, hot-pluggable, restartable, or profile-dependent? |
| D040 | IPC primitives | Open | Which native IPC mechanisms are first-class: channels, message queues, pipes, shared memory, events, wait sets, ports/endpoints? |
| D041 | Command/event/pub-sub model | Open | Should Linear provide common message-routing conventions/frameworks, or leave them entirely to user libraries? |

## G. POSIX, Rust `std`, C, and C++ contracts

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D042 | Embedded POSIX profile | Open | Precisely which POSIX APIs and semantics are guaranteed by each Linear profile? |
| D043 | Rust `std` platform profile | Open | Precisely which parts of `std` are supported (`thread`, `sync`, `time`, `io`, `fs`, `net`, `env`, `process`, etc.) and under what configuration? |
| D044 | Rust global allocator | Open | How does Rust's global allocator map onto Linear's configurable memory policies and protected domains? |
| D045 | C runtime / libc | Open | Which libc strategy and ABI should Linear target: picolibc/newlib/musl/custom compatibility layer or profile-dependent choice? |
| D046 | C++ runtime | Open | What support is expected for `std::thread`, exceptions, RTTI, static constructors, `new/delete`, atomics, TLS, and ABI/runtime services? |
| D047 | VFS / filesystem model | Open | Is VFS core, optional compatibility subsystem, or service; how do `/dev`, mounts, pseudo-filesystems, and `std::fs` relate? |
| D048 | Networking model | Open | Is TCP/IP just another composable service; how are BSD sockets, readiness, DNS/DHCP, packet buffers, and multiple interfaces exposed? |

## H. Time, power, and clocking

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D049 | Clock domains | Open | Define monotonic, realtime/calendar, CPU/runtime, and hardware clock domains and their adjustment semantics. |
| D050 | Power management | Open | Define idle, tickless operation, sleep/deep sleep, device suspend/resume, wake sources, power domains, and clock scaling. |
| D051 | Time across suspend | Open | Does monotonic time include suspended time, which timers remain active, and how are deadlines restored after wake? |
| D052 | Power dependency graph | Open | Can device/service composition express regulators, clocks, wake dependencies, and suspend ordering? |

## I. Testing, simulation, verification, and observability

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D053 | Standard test harness APIs | Open | Define first-party `FakeClock`, virtual kernel, fake IRQ/DMA/buses, fault injection, graph harness, and system harness APIs. |
| D054 | Deterministic scheduler simulation | Open | How precisely can tests control/interrogate runnable tasks, interleavings, wakeups, timers, and injected events? |
| D055 | Replay and reproducibility | Open | What event/seed/config/build metadata is captured so failing system tests can be reproduced exactly? |
| D056 | Property/model-based testing | Open | Which kernel state machines and invariants should be tested with property testing or model checking? |
| D057 | Fuzzing strategy | Open | Which boundaries should have supported host fuzz harnesses: syscalls, parsers, drivers, resolver, filesystems, networking? |
| D058 | Logging and tracing | Open | Structured logging, trace events, timestamps, per-core buffers, compile-time removal, and realtime overhead guarantees. |
| D059 | Runtime metrics / diagnostics | Open | Standardize stack high-water, heap/pool usage, queue depth, task states, IRQ counters, scheduler statistics, and driver/service health. |
| D060 | Crash dump / postmortem format | Open | What persistent crash information and symbolizable trace format should be standardized? |

## J. Build, configuration, deployment, and evolution

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D061 | Build-system boundary | Open | Cargo-first with CMake interop, neutral meta-build, or another approach; how are C/C++ and Rust packages composed? |
| D062 | Configuration model | Open | What replaces or simplifies Kconfig + devicetree + build scripts while remaining inspectable and expressive? |
| D063 | Generated-system introspection | Open | Standardize commands/artifacts for inspecting device graph, service graph, resources, memory, scheduling, and provider selection. |
| D064 | Static vs dynamic loading | Open | Are MCU profiles statically linked only; do MMU profiles eventually support independent applications/shared libraries/modules? |
| D065 | ABI/version compatibility | Open | Which interfaces are stable across releases: syscall ABI, HAL traits, service contracts, config schema, board schema, test APIs? |
| D066 | Firmware update boundary | Open | A/B images, signing, rollback, metadata, configuration migration, bootloader contract, and independent app/kernel update compatibility. |
| D067 | Reproducible builds | Open | Which toolchain/config/source metadata is required to reproduce an exact firmware image? |

## K. Security, safety, and profiles

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D068 | Security baseline | Open | Secure boot expectations, image signing, RNG/entropy, secret storage, debug policy, syscall validation, stack protections, and privilege rules. |
| D069 | Safety / deterministic profile | Open | What concrete guarantees distinguish General, RT/Deterministic, and Safety profiles? |
| D070 | Freedom from interference | Open | If safety/protection profiles exist, how are memory, CPU time, interrupts, devices, and shared resources isolated between components? |
| D071 | Certification-oriented constraints | Open | Which architecture choices should preserve a path toward IEC 62304, IEC 61508, ISO 26262, or similar assurance regimes without making certification a v1 requirement? |
| D072 | Runtime configurability in safety profiles | Open | Which dynamic behaviors—allocation, service rebinding, hot-plug, loading, runtime configuration—must be prohibited or bounded in stricter profiles? |

## L. Portability, ecosystem, and scope

| ID | Topic | Status | Question to settle |
|---|---|---|---|
| D073 | Architecture-port contract | Open | What minimal port must an architecture implement: boot, context switch, IRQ control, atomics, monotonic timer, memory map, optional MPU/MMU/SMP hooks? |
| D074 | Board-support-package contract | Open | What belongs in architecture, SoC, board, device, and product layers? |
| D075 | Out-of-tree ecosystem | Open | How are third-party HALs, drivers, services, board packages, and proprietary components integrated without patching Linear? |
| D076 | Package/version policy | Open | How are public crates/libraries, C headers, schemas, driver/service packages, and compatibility ranges versioned? |
| D077 | Licensing policy | Open | What licensing strategy enables commercial/proprietary applications while encouraging reusable drivers and ecosystem contributions? |
| D078 | Explicit non-goals | Open | Which features are intentionally outside Linear's scope so architecture does not grow into a small Linux clone? |

---

# Accepted decisions and rationale

## D001 — Concurrency policy stays above the kernel

**Status:** Accepted

### Decision

Linear provides general-purpose mechanisms such as threads, mutexes/condition variables, semaphores, bounded queues/channels, events, timers, wait sets, priority-aware scheduling, and ISR-safe signaling.

It does **not** make one application concurrency model fundamental to the RTOS. Applications may choose ordinary blocking threads, Active Objects/state machines, an optional Rust async executor such as Embassy, or another model.

### Rationale

Trying to make blocking and Rust async interfaces look identical introduces async coloring and substantial machinery. Stackful fibers or injected runtimes would also work against Linear's goal of supporting simple ordinary `std` code. The kernel should provide mechanisms while concurrency policy remains above it.

---

## D002 — Active Objects are a library pattern

**Status:** Accepted

### Decision

Active Objects are supported cleanly by building them from existing OS facilities such as threads, channels/message queues, timers, events, and priorities. AO is not a kernel object and is not required by the service model.

A user/framework may implement one AO per RTOS thread, many AOs on one run-to-completion worker, or many AOs on a worker pool without changing the kernel ABI.

### Rationale

AO localizes asynchronous behavior at message boundaries while keeping event handlers and domain logic ordinary synchronous code. Keeping AO above the kernel preserves runtime neutrality and makes component logic easy to test without its production runtime.

---

## D003 — POSIX is a compatibility profile

**Status:** Accepted

### Decision

Linear aims for a useful embedded POSIX profile and practical Rust `std` support, but does not model its internal architecture around Unix abstractions. Native typed service/device interfaces may coexist with POSIX file descriptors, `/dev`, sockets, pthreads, and libc.

### Rationale

POSIX is valuable for portable software and makes Rust `std` platform integration more practical. Full Unix process/session/signal semantics should not distort MCU-oriented architecture where they provide little value.

---

## D004 — Testability through substitution is a core principle

**Status:** Accepted

### Decision

> Every externally observable dependency boundary should also be a test seam where practical.

Production dependencies must be replaceable by compatible test implementations without modifying the code under test. Linear should support pure unit tests, component/service tests, driver tests with fake HAL/device models, partial service graphs, complete host userspace graphs, deterministic kernel tests, simulated/emulated integration, hardware integration, and full product tests.

### Rationale

The same abstractions used for portability and composition should enable modular testing. Behavioral fakes and device models are preferred over brittle call-count mocks when realistic semantics are more useful. Fault injection is first-class.

---

## D005 — OS time is virtualizable and controllable under test

**Status:** Accepted

### Decision

Kernel timekeeping must not be intrinsically tied to the physical timer peripheral. The timer subsystem consumes an abstract monotonic source whose production implementation is hardware-backed and whose test implementation can use virtual time.

The virtual environment must be capable of controlling OS facilities derived from monotonic time, including Rust `Instant`, thread sleep, POSIX monotonic clocks, timed waits, poll/socket timeouts, kernel/AO timers, and simulated IRQ/DMA/device events.

Desired test modes include real time, scaled time, manual deterministic time, and next-event/auto-advance discrete-event time. Realtime/calendar time remains logically separate from monotonic time.

### Rationale

Normal application code should not require explicit clock injection merely to make ordinary `std` sleeps and OS timeouts testable. Built-in virtual OS time enables complete userspace/system tests to execute long-duration behavior rapidly and deterministically.

### Open sub-details

Tracked mainly under D023, D049, D051, D053, and D054: periodic timer catch-up semantics, simultaneous event ordering, clock-domain semantics, suspend behavior, and exact test-controller APIs.

---

## D006 — Dynamic allocation is enabled by default

**Status:** Accepted

### Decision

The default Linear configuration provides ordinary dynamic allocation so portable Rust `std`, C, C++, networking, filesystems, and third-party libraries remain ergonomic.

Heap availability is a **product/profile configuration choice**, not something service and application interfaces should encode into type signatures. Linear should avoid spreading allocator generics through ordinary APIs solely to support stricter builds later.

Special memory semantics remain explicit—for example DMA-safe memory, non-cacheable/coherent memory, TCM, retention RAM, fixed packet pools, and special realtime pools.

### Rationale

Making `std` a first-class application goal conflicts with treating dynamic allocation as universally forbidden. The architecture should preserve normal language ergonomics while allowing products with stronger determinism requirements to tighten policy through configuration.

---

## D007 — Kernel allocation policy is profile-driven, not universally static

**Status:** Accepted

### Decision

Linear will **not** impose a universal safety-RTOS rule that the kernel can never use a general-purpose heap at runtime.

The default/general profile follows the more flexible Zephyr/NuttX design family: userspace heap enabled by default, kernel dynamic allocation permitted when configured, runtime object creation supported, and fixed pools/slabs/static allocation available.

Stricter profiles must be able to tighten the same architecture without changing application/service APIs. An RT/Deterministic profile may require bounded pools in critical paths and auditable allocation policy. A Safety profile may disable/restrict kernel runtime heap allocation, statically provision kernel objects or use bounded pools, restrict userspace allocation after initialization, and explicitly declare/validate resource maxima.

### Rationale

Current RTOSes span a spectrum: NuttX strongly favors POSIX flexibility; Zephyr provides configurable heaps and slabs; FreeRTOS supports dynamic and static object creation; SAFERTOS represents the strict safety-oriented end with tightly controlled memory. Linear intentionally starts closer to Zephyr/NuttX while preserving a clean path toward stronger reliability and medical/safety-oriented profiles.

### API principle

Allocation policy should normally remain below application APIs. Avoid parallel `create()` versus `create_static()` application models unless the semantics genuinely differ.

### Observability requirement

Even General builds should expose heap current/peak usage, allocation failures, fragmentation information where meaningful, pool utilization, per-domain usage, stack high-water marks, and special-memory usage. Build/composition tooling should report statically known memory budgets where possible.

### Open sub-details

Tracked mainly under D012-D015, D044, D059, D069, and D072: allocator implementation, memory domains, stack allocation, post-init controls, Rust allocator integration, metrics, and exact safety-profile guarantees.

---

# Current decision sequence

We will work through open decisions interactively rather than attempting to settle the entire backlog at once.

## Next: D008 — Memory protection and execution domains

**Status:** Open

Determine whether Linear should architecturally support flat, MPU-protected, and MMU/process execution domains from the beginning, whether they are profiles of one kernel architecture, and whether the kernel/userspace boundary should compile down to direct calls in flat configurations.

No decision has been made yet.
