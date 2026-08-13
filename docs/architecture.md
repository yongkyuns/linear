# Linear RTOS Architecture

**Working draft v0.2**

> Linear is an experimental embedded RTOS architecture focused on simple, reusable application code across MCU targets. The current direction combines portable C/C++ and Rust `std`, an embedded POSIX compatibility profile, capability-oriented drivers, declarative device/service composition, and concurrency-neutral kernel mechanisms.

## 1. Core goals

- Keep ordinary application code independent from concrete hardware.
- Reuse drivers across MCU/SoC families.
- Avoid repetitive board-specific bring-up glue.
- Support C/C++ and Rust `std` applications where practical.
- Treat POSIX as a compatibility profile, not as the internal ontology of the kernel.
- Keep concurrency policy out of the kernel: threads, Active Objects, state machines, and Rust async are application/framework choices.
- Prefer static composition and allocation where practical.
- Make host simulation/testing a first-class use case.

## 2. Architectural layers

```text
Application
  |
  | logical services / capabilities
  v
Service layer
  | Navigation, Motion, Location, Storage, Network, Logging, ...
  v
Device-class layer
  | Sensor, CAN, UART, BlockDevice, GPIO, ...
  v
Driver / HAL capability layer
  | I2c, SpiBus, SpiDevice, Pin, DMA, Interrupt, Clock, ...
  v
Target HAL / BSP
  v
Hardware
```

Ordinary applications should not know which sensor, bus, pin, or MCU is present.

## 3. What to borrow

| Source | Borrow | Avoid |
|---|---|---|
| NuttX | POSIX-facing environment, libc/std friendliness, `/dev`, familiar upper/lower separation | repetitive board-specific bring-up; allowing POSIX to dictate all internals |
| Zephyr | declarative hardware graph, bindings, automatic instance creation, overlays | macro/build-system complexity |
| `embedded-hal` | small capability-oriented interfaces, reusable device drivers | assuming HAL portability alone solves whole-system composition |
| Embassy | ownership-aware HAL ideas and optional lightweight async execution | making async part of core service/driver contracts |
| Active Objects | private state, message-driven concurrency, run-to-completion handlers | making AO a mandatory kernel concept |

## 4. Capability-oriented HAL

The HAL should expose small interfaces such as:

```text
I2c
SpiBus
SpiDevice
InputPin
OutputPin
Pwm
Delay
DmaChannel
InterruptSource
ClockSource
```

A reusable device driver should require only the capabilities it actually needs. A BMI270 driver, for example, should depend on a `SpiDevice` or `I2c` plus optional interrupt/reset capabilities, not on STM32/NXP/ESP-specific types.

Native target HALs may expose richer extensions while also satisfying the portable interfaces.

### Shared buses

Physical buses and logical device endpoints should be distinct:

```text
SPI controller
     |
 shared bus / arbitration
   /      |       \
 IMU     ADC      Flash
  |       |         |
 CS1     CS2       CS3
```

## 5. Device graph

Driver portability alone does not remove board glue. Linear should have a declarative device/resource graph that can:

- describe controllers, pins, IRQs, DMA channels, clocks, regulators, buses, and attached devices;
- bind driver implementations to compatible hardware;
- describe dependencies and configuration;
- construct devices in dependency order;
- reject resource conflicts at build time where possible;
- support board/product overlays without copying bring-up code.

The syntax is still open: devicetree-compatible, a neutral schema, or a Rust-native DSL are all candidates.

Example Rust-like idea:

```rust
board! {
    spi1: Spi {
        peripheral: SPI1,
        sck: PA5,
        miso: PA6,
        mosi: PA7,
    },

    imu: Bmi270 {
        bus: spi1,
        cs: PA4,
        irq: PB0,
    },
}
```

## 6. Device graph vs service graph

Physical hardware and logical application capabilities should be modeled separately.

```text
Hardware/device graph                  Service graph

BMI270                                 Navigation
  |                                       |
 SPI                                    Motion
  |                                       |
GPIO                                  ImuSource
                                          |
                                        BMI270
```

The device graph answers **what hardware exists and how is it connected?**

The service graph answers **what capabilities does the application need and which providers satisfy them?**

## 7. Applications consume services, not devices

Normal application code should ask for capabilities such as:

```text
Motion
Location
Network
Storage
Clock
Logging
Navigation
FirmwareUpdate
VehicleSpeed
Pose
Camera
```

It should not normally ask for:

```text
BMI270
SPI1
USART2
GPIOA4
u-blox M10
```

Example composition:

```text
Application
   |
   +-- Navigation
   |     +-- Motion
   |     |    +-- ImuSource
   |     +-- Location
   |     +-- optional VehicleSpeed
   |
   +-- Telemetry
         +-- Network
         +-- optional Storage
```

A service may declare required and optional capabilities:

```text
Navigation provides:
    navigation.pose
    navigation.velocity

Navigation requires:
    motion.imu
    location.gnss

Navigation optional:
    vehicle.wheel_speed
    heading.absolute
```

Provider selection belongs to system/product composition rather than application source code.

## 8. Services are not tasks or runtimes

A service may be:

- a pure library;
- a stateful object;
- an interrupt-driven provider;
- a dedicated thread;
- several worker threads;
- an Active Object;
- a state machine;
- an async task;
- a kernel facility.

Do not assume:

```text
service = thread
service = async task
service = active object
```

Service composition and scheduling policy should remain independent.

## 9. Concurrency: mechanisms in the RTOS, policy above it

Blocking and async application APIs are fundamentally different, especially in Rust. Linear should not try to hide this with a universal async signature, stackful-fiber trick, or injected runtime that contaminates ordinary APIs.

The kernel should instead provide general mechanisms:

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

Higher-level models are libraries:

```text
                         APPLICATIONS

          ordinary threads       Active Objects        Rust async
          C/C++ / Rust std       / state machines      optional
                |                       |                  |
                |                 optional library    Embassy/etc.
                |                       |                  |
                +-----------+-----------+------------------+
                            |
                  std / POSIX / native APIs
                            |
                 GENERAL RTOS MECHANISMS
          threads / queues / events / timers / wait
```

### Blocking applications

Ordinary code can simply block a thread:

```rust
fn logger() {
    loop {
        let record = queue.recv();
        storage.write(&record);
    }
}
```

When `recv()` blocks, the RTOS schedules another runnable thread. No language-level async is required.

### Optional non-blocking/waitable primitives

Where useful, low-level APIs may expose:

```text
try_recv()
try_send()
start_transfer()
status()
readable event
writable event
completion event
timer event
```

These allow event-driven libraries to be built without making the core service traits async.

### Optional Rust async

Async should be an opt-in library layer:

```text
OS queues / waitables / timers
             |
       Future adapters
             |
     embassy-executor
             |
       async application
```

Embassy should not be required by the kernel, HAL, drivers, or service contracts.

## 10. Active Objects as a library pattern

Active Objects are attractive for reactive embedded applications because they localize asynchronous behavior at message boundaries while allowing implementation code to remain ordinary synchronous Rust.

Example:

```rust
struct Navigation {
    filter: FusionFilter,
    state: NavState,
}

enum NavEvent {
    Imu(ImuSample),
    Gnss(GnssFix),
    WheelSpeed(f32),
    Reset,
}

fn run(mut nav: Navigation, rx: Receiver<NavEvent>) {
    while let Ok(event) = rx.recv() {
        nav.dispatch(event);
    }
}
```

Handlers remain ordinary functions:

```rust
fn on_imu(&mut self, sample: ImuSample) {
    self.filter.update(sample);
    self.update_state();
}
```

AO should **not** be a kernel primitive. Users or an optional framework should be able to build it from existing OS facilities:

```text
thread
bounded queue/channel
event notification
timer
priority
wait primitives
ISR-safe post/signal
```

Possible AO scheduling policies include:

```text
1 AO = 1 RTOS thread
many AOs = 1 run-to-completion worker
many AOs = N-worker pool
```

The policy belongs to the AO library/application, not the kernel ABI.

## 11. POSIX compatibility and Rust `std`

Full POSIX conformance is not the objective. The preferred goal is a **first-class embedded POSIX compatibility profile plus native OS APIs where they are cleaner**.

High-value POSIX areas likely include:

```text
pthread threads
mutexes / condition variables
semaphores

clock_gettime
sleep / nanosleep

open / close
read / write
fcntl / ioctl

poll / select

basic filesystem APIs
BSD sockets
errno
```

Potentially unnecessary or optional on MCU-class targets:

```text
fork
full exec/process model
process groups
job control
terminal sessions
Unix users/groups
full signal semantics
```

POSIX should be an adapter/profile, not the kernel architecture:

```text
                     Applications
                 /                   \
          POSIX / libc             Native API
          Rust std / C++       typed services/devices
                 \                   /
                  \                 /
                 kernel mechanisms
          threads / queues / wait / timers
                         |
                 services / drivers
```

Rust `std` support is a target/runtime integration problem. POSIX-like semantics make a port easier but should not dictate the kernel design.

Desired user-facing support, where practical:

```rust
std::thread
std::sync
std::time
std::fs
std::net
```

## 12. Product composition

An application should declare the services it wants. The resolver pulls in transitive requirements and omits unused pieces.

```yaml
application: vehicle-recorder

services:
  - navigation
  - telemetry
  - diagnostics

features:
  storage: optional
  wheel_speed: optional
```

Conceptually:

```text
Application manifest
      +
Service manifests
      +
Board/device description
      +
Product configuration
      |
      v
Composition resolver
      |
      v
Validated device graph + service graph
      |
      v
Generated construction / registration
```

The resolver should be orthogonal to execution policy: it decides what exists, not whether a service is a thread, AO, state machine, or async task.

## 13. POSIX-visible devices vs logical services

Both may be useful:

```text
Physical/device view             Logical/service view

/dev/ttyS2                       Network
/dev/spi1                        Location
/dev/imu0                        Motion
/dev/can0                        VehicleSpeed
/dev/mmcsd0                      Storage
```

The open question is which layer owns each abstraction and when a service/provider should expose a POSIX endpoint.

## 14. Host simulation and testing

Host execution should be first-class:

```text
Embedded target                  Host test

Motion -> BMI270                 Motion -> recorded IMU
Location -> u-blox               Location -> recorded GNSS
Storage -> NOR flash             Storage -> host filesystem
Clock -> hardware timer          Clock -> simulated clock
```

Desired properties:

- service contracts are mockable without reproducing low-level HAL behavior;
- the same composition system can select embedded or host providers;
- deterministic/simulated time is available for repeatable tests;
- fault injection is possible at service and device boundaries.

## 15. Configuration concerns

Keep separate concerns separate:

| Concern | Examples |
|---|---|
| Hardware topology | controllers, pins, devices, IRQ/DMA channels, regulators |
| Driver selection | implementation bound to a compatible device |
| Service selection | navigation, logging, networking, storage |
| Provider binding | which `Location` provider satisfies the capability |
| Software features | protocols, filesystem formats, diagnostics |
| Resource policy | stacks, queue depth, pools, priorities, deadlines |
| Product overlay | variant-specific differences without application forks |

A major goal is to avoid recreating Zephyr's Kconfig + DTS + CMake complexity in a different form.

## 16. Resource ownership and lifetime

Open requirements:

- exclusive hardware resources should not be instantiated twice;
- shared resources require explicit arbitration/management;
- driver lifetime must be defined for static and optional runtime devices;
- application handles need stable lifetime semantics;
- Rust ownership and a stable C ABI must coexist safely;
- DMA buffers/cache coherency/zero-copy ownership need an explicit model.

## 17. Initialization and dependency ordering

Initialization should follow the resolved dependency graph:

```text
regulator
    ->
bus controller
    ->
device driver
    ->
device-class service
    ->
higher service
    ->
application
```

Open questions include failure propagation, optional providers, restart/late availability, and whether MCU targets need hot-plug at all.

## 18. Scheduling and real-time behavior

The scheduler should provide predictable primitives without embedding a higher-level application framework.

Areas to define:

- thread priorities and scheduling classes;
- priority inheritance/ceiling;
- bounded queues and deterministic backpressure;
- ISR-to-thread/event wakeup latency;
- timers/timeouts;
- stack sizing and diagnostics;
- SMP affinity/pinning if applicable.

Higher-level runtimes map onto these mechanisms:

```text
AO framework:
    queues + worker thread(s) + priorities

Embassy integration:
    one or more RTOS threads hosting executor(s)

traditional app:
    ordinary RTOS/POSIX threads
```

## 19. Current design principles

1. **Mechanisms, not concurrency policy.** Kernel supplies threads, queues, synchronization, timers, events, and waiting.
2. **Do not make async part of core service contracts.** Async/Future types stay in opt-in libraries.
3. **Applications depend on capabilities.** Hardware details should rarely appear in normal application code.
4. **Two graphs, not one.** Device composition and logical service composition are distinct.
5. **Services are not schedulable entities by definition.**
6. **Active Objects are a library pattern, not a kernel feature.**
7. **POSIX compatibility is a profile.** Support the useful subset without making Unix semantics internal architecture.
8. **Static by default, dynamic by need.**
9. **Portable core, native escape hatches.**
10. **Host execution is first-class.**

## 20. Open design questions

1. What exact POSIX subset is required for the intended C/C++ ecosystem and practical Rust `std`?
2. Which kernel primitives should be first-class: threads, mutexes, condvars, semaphores, queues, events, wait sets, pipes, message queues?
3. What semantics are required for bounded, static, ISR-safe channels?
4. Should `poll()`/`select()` be backed by one general wait-set/event primitive?
5. Which device operations need completion objects versus simple blocking/non-blocking methods?
6. What should optional async support adapt, and what should stay deliberately synchronous?
7. What is the cleanest optional Embassy integration?
8. What should an optional first-party AO crate provide versus leave to users?
9. Should AO support thread-per-object, shared run-to-completion workers, or multiple policies?
10. How should commands, events, and pub/sub be composed without creating a global event-bus anti-pattern?
11. How should service contracts cross Rust/C/C++ boundaries?
12. How are providers discovered and selected?
13. How are optional capabilities represented cleanly?
14. What board-description format should be used?
15. How much ownership validation belongs in Rust types versus the composition resolver?
16. How are DMA buffers, cache coherency, and zero-copy ownership modeled?
17. How should native handles, POSIX file descriptors, service handles, sockets, and device nodes relate?
18. What memory-allocation profiles should be supported?
19. How can configuration remain inspectable and significantly simpler than Zephyr's combined machinery?
20. What should the workflow be for adding a board, driver, service provider, AO component, and application?

## 21. Suggested design passes

### A. Kernel mechanisms
Define the minimal thread, synchronization, queue/channel, event, timer, and wait-set primitives.

### B. POSIX / Rust `std` profile
Identify the smallest useful POSIX surface and map libc/Rust `std` requirements onto it.

### C. Driver capability model
Define reusable HAL/device interfaces and shared-resource behavior without depending on async.

### D. Device/resource resolver
Define board/device description, ownership validation, dependency ordering, and product overlays.

### E. Service contracts and composition
Define capabilities, provider selection, lifecycle, and language boundaries independently of scheduling.

### F. Active Object prototype
Implement a small optional AO library using only standard OS mechanisms: bounded channels, threads/workers, timers, and run-to-completion dispatch.

### G. Optional Embassy prototype
Adapt OS queues/timers/waitables into `Future`s without changing kernel, driver, or service interfaces.

### H. Representative applications
Build a minimal sensor logger, a navigation/telemetry product, and a host simulation using the same service logic with different concurrency choices.

## 22. Concrete architecture experiments

| Prototype | Question | Success criterion |
|---|---|---|
| POSIX/Rust `std` smoke test | What minimum OS surface is needed? | representative `std::thread`, `std::sync`, `std::net`, `std::fs` subset runs |
| Shared SPI + IMU/Flash | Can ownership, arbitration, DMA and priorities remain composable? | no board-specific application glue; deterministic behavior |
| Service graph resolver | Can `Navigation` be requested independently of physical provider choices? | same application logic builds for two boards and a host |
| AO navigation pipeline | Can reactive components use only threads + bounded channels? | no async in navigation/filter/service core |
| Shared AO dispatcher | Can several AOs share workers safely? | lower stack cost while preserving run-to-completion |
| Embassy adapter | Can OS waits/channels/timers become Futures without contaminating core interfaces? | async sample runs; no Embassy dependency below adapter |
| Optional capability variant | Can wheel speed/heading augment Navigation cleanly? | no application/service fork |

## 23. Non-goals / cautions

- Do not recreate Linux driver complexity without a concrete need.
- Do not pursue full POSIX merely for standards purity.
- Do not make every abstraction dynamically discoverable because POSIX uses runtime handles.
- Do not make Rust async, Embassy, AO, actors, or another runtime mandatory.
- Do not invent a stackful/fiber runtime solely to hide async coloring.
- Do not expose raw hardware resources to ordinary apps merely because the HAL permits it.
- Do not make composition/configuration too magical to inspect.
- Do not make the service graph depend on scheduling policy.

## 24. Working thesis

> Linear should be a small RTOS that provides conventional threads, synchronization, bounded messaging, timers, waiting, and a useful POSIX compatibility profile for portable C/C++ and Rust `std`; uses capability-oriented abstractions and declarative composition for reusable drivers/hardware; and keeps application concurrency policy outside the kernel. Services are composed independently of whether applications use ordinary threads, Active Objects/state machines, or an optional Rust async runtime such as Embassy.

The architecture should optimize for **simple ordinary code first**:

```text
kernel mechanisms
      |
std / POSIX / native wrappers
      |
+----------------+----------------+----------------+
|                |                |                |
threads        AO library      async library     custom model
```

This thesis remains provisional until the POSIX/std profile, kernel messaging/wait primitives, service contracts, device resolver, AO prototype, and optional async adapter have been measured on representative MCU-class targets.
