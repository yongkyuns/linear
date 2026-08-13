# Linear RTOS Architecture

**Working draft v0.3**

> Linear is an experimental embedded RTOS architecture focused on simple, reusable, and testable application code across MCU targets. The current direction combines portable C/C++ and Rust `std`, an embedded POSIX compatibility profile, capability-oriented drivers, declarative device/service composition, concurrency-neutral kernel mechanisms, and deterministic testability from individual components through complete systems.

## 1. Core goals

- Keep ordinary application code independent from concrete hardware.
- Reuse drivers across MCU/SoC families.
- Avoid repetitive board-specific bring-up glue.
- Support C/C++ and Rust `std` applications where practical.
- Treat POSIX as a compatibility profile, not as the internal ontology of the kernel.
- Keep concurrency policy out of the kernel: threads, Active Objects, state machines, and Rust async are application/framework choices.
- Prefer static composition and allocation where practical.
- Make host execution and simulation first-class.
- Make **testability through substitution** a fundamental architectural property.
- Allow applications to be executed and tested as individual components, partial service graphs, complete userspace graphs, or full target systems.
- Make OS time and scheduling controllable under test so long-duration and concurrency-sensitive behavior can be tested deterministically and faster than real time.

### Testability principle

> **Every dependency boundary should also be a test seam.** Hardware, services, external inputs, time, I/O, and — where practical — scheduling should have replaceable or controllable boundaries.

A good Linear component should have explicit dependencies, no hidden hardware assumptions, minimal coupling between domain logic and execution/runtime glue, deterministic inputs where practical, and the ability to run independently of the complete product.

---

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

Each abstraction boundary should support production and test implementations where that is meaningful.

```text
Production                       Test

STM32 SPI                        Fake SPI / simulated bus
BMI270                           simulated BMI270
u-blox GNSS                      recorded/simulated GNSS
system monotonic clock           virtual monotonic clock
network                          fake network
flash/filesystem                 memory/host-backed storage
hardware IRQ                     injected event/IRQ
```

---

## 3. What to borrow

| Source | Borrow | Avoid |
|---|---|---|
| NuttX | POSIX-facing environment, libc/std friendliness, `/dev`, familiar upper/lower separation | repetitive board-specific bring-up; allowing POSIX to dictate all internals |
| Zephyr | declarative hardware graph, bindings, automatic instance creation, overlays | macro/build-system complexity |
| `embedded-hal` | small capability-oriented interfaces, reusable device drivers | assuming HAL portability alone solves whole-system composition |
| Embassy | ownership-aware HAL ideas and optional lightweight async execution | making async part of core service/driver contracts |
| Active Objects | private state, message-driven concurrency, run-to-completion handlers | making AO a mandatory kernel concept |
| Host simulation/model-based testing | deterministic inputs, virtual devices, replay, fault injection | requiring instruction-level emulation for tests that only need behavioral simulation |

---

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

### HAL test implementations

Portable HAL contracts should support several kinds of test doubles:

- **mock** — verifies calls/transactions;
- **fake** — provides a useful behavioral implementation;
- **device model** — simulates actual peripheral/device semantics;
- **fault-injecting provider** — produces timeouts, bus errors, delayed completion, partial failure, etc.

Tests should usually prefer behavioral fakes/models where implementation-call verification would make the test unnecessarily brittle.

Example conceptual controls:

```rust
spi.fail_next(Error::Timeout);
irq.trigger();
dma.complete();
uart.inject_rx(data);
```

---

## 5. Device graph

Driver portability alone does not remove board glue. Linear should have a declarative device/resource graph that can:

- describe controllers, pins, IRQs, DMA channels, clocks, regulators, buses, and attached devices;
- bind driver implementations to compatible hardware;
- describe dependencies and configuration;
- construct devices in dependency order;
- reject resource conflicts at build time where possible;
- support board/product overlays without copying bring-up code;
- substitute test/simulation providers without changing the code under test.

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

A test configuration should be able to replace the bottom of this graph:

```text
Production board                Test board

STM32 SPI1                      FakeSpi
   |                               |
BMI270                          SimulatedBMI270

UART2                           ReplayUart
   |                               |
u-blox GNSS                    recorded GNSS stream
```

---

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

The separation is also a testing mechanism: a test may retain the real service graph while replacing only the device graph, or replace upstream services with replay/fake providers while testing a downstream service.

---

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

### Application/service test seams

The same dependency contract should permit:

```text
Production                       Test

Motion -> BMI270                 Motion -> recorded IMU
Location -> GNSS                 Location -> recorded GNSS
Storage -> NOR                   Storage -> memory/host FS
Network -> LTE                   Network -> fake server
Clock -> kernel clock            Clock -> virtual/test clock
```

Avoid hidden global dependencies where they make substitution difficult. Dependencies may be statically resolved or type-erased, but they should remain visible to the composition system.

---

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

This separation is also important for testing. The core logic of a service should normally be runnable without starting its production thread/runtime wrapper.

---

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

---

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

### Testing Active Objects

AO logic should be testable without its runtime wrapper:

```rust
let mut nav = Navigation::new(...);

nav.dispatch(Event::Imu(sample));
nav.dispatch(Event::Gnss(fix));
nav.dispatch(Event::Tick);

assert_eq!(nav.state(), expected);
```

Tests of navigation behavior should not require spawning a thread, sleeping, or waiting for scheduling races. The AO runtime itself is tested separately for queueing, ordering, priorities, overflow, shutdown, and wakeup behavior.

---

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

Ordinary `std` code should participate in system-level testing. When `std::time::Instant`, `std::thread::sleep`, timed condvars, sockets, etc. are backed by Linear's OS services, a virtualized Linear time domain should allow them to run deterministically under system tests without application-specific clock injection.

---

## 12. Product composition and test composition

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

### Partial graph execution

A central testing requirement is the ability to instantiate arbitrary useful slices of an application graph.

Given:

```text
IMU -> Motion -> Navigation -> Telemetry -> Network
```

users should be able to test:

```text
IMU -> Motion
```

or:

```text
RecordedMotion -> Navigation
```

or:

```text
RecordedNavigation -> Telemetry -> FakeNetwork
```

or the complete userspace graph:

```text
FakeIMU
  |
Motion
  |
Navigation
  |
Telemetry
  |
FakeNetwork
```

Test composition should ideally use the same resolver as production composition.

Example conceptual test description:

```yaml
test: navigation-replay

services:
  - navigation

bindings:
  motion: recorded-imu
  location: recorded-gnss
  clock: virtual-clock
```

---

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

Both views should remain testable through the same underlying provider substitutions where possible.

---

## 14. Test architecture

Testability is not a separate tool layered on after implementation. It should influence interface boundaries, composition, timekeeping, kernel structure, and target backends.

### 14.1 Test pyramid

Linear should explicitly support multiple test granularities:

| Level | Scope | Typical environment |
|---|---|---|
| L0 | pure algorithms/data structures | host |
| L1 | component/service logic | host + fake dependencies |
| L2 | driver logic | host + fake HAL/device model |
| L3 | partial service graph | host |
| L4 | complete userspace graph | host |
| L5 | kernel mechanisms | deterministic host tests |
| L6 | kernel + userspace | host simulation/emulated target |
| L7 | architecture/HAL integration | physical board/emulator |
| L8 | full product/system | physical product |

The majority of tests should live near the bottom of the pyramid and run quickly without physical hardware.

### 14.2 Pure logic first

Application algorithms should be ordinary libraries whenever possible:

```rust
struct Fusion { ... }

impl Fusion {
    fn update_imu(&mut self, x: ImuSample) { ... }
    fn update_gnss(&mut self, x: GnssFix) { ... }
    fn pose(&self) -> Pose { ... }
}
```

These tests should need no RTOS, thread, queue, HAL, or async runtime.

### 14.3 Substitute capabilities rather than adding test backdoors

Testing should normally use alternative implementations of the same contracts rather than `#[cfg(test)]` branches embedded through production code.

```text
real provider <-> fake provider <-> replay provider <-> simulator
```

The composition mechanism used for product variation should also be the primary dependency-injection mechanism for tests.

### 14.4 Mock, fake, replay, and simulation

Linear should support different styles because they answer different questions:

- **mock**: did expected operations occur?
- **fake**: does the component work against a simple functioning implementation?
- **replay**: does it reproduce behavior from recorded real-world data?
- **simulation/device model**: does it interact correctly with a behavioral model?

No one approach should be mandatory.

### 14.5 Fault injection

Failure control should be built into testing providers. Representative scenarios include:

```text
SPI timeout
I2C NACK
DMA late completion
partial storage write
GNSS outage
CAN bus-off
queue full
allocation failure
network disconnect
interrupt at a specific scheduling point
```

Fault injection should be deterministic and scriptable where possible.

### 14.6 Interface conformance suites

Important interfaces should have reusable behavioral test suites.

For example, every SPI implementation might run common tests for:

```text
basic transfer
CS semantics
transaction boundaries
multiple sizes
concurrent/shared access
timeout/error behavior
DMA path where supported
```

The same concept can apply to storage, channels, clocks, timers, sockets, and service-provider contracts.

---

## 15. Virtual and controllable OS time

Time should be treated as a dependency of the operating system, not as an immutable property of the hardware.

Linear's kernel timer logic should depend on an abstract monotonic time source:

```text
                    Kernel time subsystem
                           |
                   monotonic time source
                    /              \
                   /                \
          HardwareClock          VirtualClock
               |                     |
          timer peripheral      test controller
```

### 15.1 Separate timekeeping from timer processing

Avoid designing kernel timers around a mandatory periodic hardware tick such as:

```text
1 ms IRQ -> ticks++ -> scan timers
```

Prefer a deadline-oriented design:

```text
TimeSource::now()
        |
    TimerQueue
        |
 next deadline
```

Production:

```text
next deadline
     |
program hardware compare
     |
IRQ
     |
expire timer
```

Test:

```text
next deadline
     |
advance virtual time directly
     |
expire timer
```

The timer queue and timeout semantics remain common.

### 15.2 OS services that should share the controllable monotonic domain

Where applicable:

```text
clock_gettime(CLOCK_MONOTONIC)
nanosleep / sleep
std::time::Instant
std::thread::sleep
pthread_cond_timedwait
sem_timedwait
poll/select timeout
socket timeout
message queue/channel timeout
kernel timers
AO timers
scheduler timeouts
```

This gives whole-system tests a much stronger capability than application-level dependency-injected clocks alone.

### 15.3 Time modes

Test/runtime infrastructure should consider at least:

1. **real time** — normal target behavior;
2. **scaled time** — e.g. 10x or 100x for interactive/system tests;
3. **manual deterministic time** — time advances only under test control;
4. **auto-advance/discrete-event mode** — when no task can make progress, jump directly to the next scheduled event/deadline.

Conceptual test API:

```rust
system.advance(Duration::from_secs(10));
system.run_until_idle();
system.run_next_event();
system.run_until(deadline);
system.run_for(Duration::from_hours(24));
```

A simulated 24-hour device scenario should not require 24 hours of host wall-clock time.

### 15.4 Monotonic vs realtime

At minimum, keep these concepts distinct:

```text
MONOTONIC
  never moves backward
  deadlines/timeouts use this

REALTIME
  wall/calendar time
  may be synchronized or adjusted
```

Tests should be able to advance monotonic time and independently set/adjust realtime. Changing realtime must not invalidate monotonic timers.

### 15.5 Simulated devices share the virtual timeline

Device models should be able to schedule events into the same deterministic time domain:

```text
t = 0.000000   SPI transfer starts
t = 0.000010   DMA completion
t = 0.005000   IMU data-ready interrupt
t = 0.010000   next IMU interrupt
```

This allows kernel timers, fake hardware, external replay, and application behavior to execute in one deterministic discrete-event simulation.

### 15.6 Time-jump semantics

The behavior of periodic timers when virtual time jumps must be explicit. Possible policies include:

```text
CatchUp      -> deliver every missed logical firing
SkipMissed   -> resume from the next future period
Coalesce     -> deliver a single indication that one or more periods elapsed
```

Ordering of simultaneous events (timer expiry, simulated IRQ, external event) must also be deterministic and documented.

### 15.7 Production cost

Physical target builds should not need to include simulation control machinery. Static composition should allow production to bind directly to a hardware timer backend while tests bind to virtual time.

---

## 16. Deterministic kernel testing

Kernel testing requires additional seams beyond user-space dependency injection.

Linear should separate **portable kernel policy/state machines** from the thin architecture-specific mechanisms that perform actual context switching and hardware control.

### 16.1 Kernel logic that should be host-testable where practical

```text
scheduler policy
run queues
wait queues
timer queues
bounded IPC/channels
priority inheritance
handle/object tables
resource ownership
wake-up state transitions
timeout/cancellation logic
```

Example scheduler test:

```rust
let low = sched.create_task(priority(1));
let high = sched.create_task(priority(10));

sched.make_ready(low);
assert_eq!(sched.next(), low);

sched.make_ready(high);
assert_eq!(sched.next(), high);
```

Priority inversion scenarios should likewise be representable as deterministic state-transition tests without booting an MCU.

### 16.2 Architecture-specific code

The following generally still require emulator/target validation:

```text
context-switch assembly
interrupt entry/exit
atomic/memory-order assumptions
MPU/MMU programming
cache maintenance
real timer programming
DMA/cache coherency
startup/linker behavior
```

Keep this layer narrow so the expensive hardware-test surface remains small.

### 16.3 Deterministic scheduler/system simulation

Linear should make it possible to drive kernel-visible events deterministically:

```text
t=0   task A runs
t=5   IRQ occurs
t=8   timer expires
t=10  DMA completes
```

A test controller should be able to control task progress, injected interrupts, timer advancement, and simulated device completion.

This permits precise testing of race-prone scenarios and may later enable systematic interleaving exploration/model checking for small cases.

### 16.4 No sleep-based synchronization in tests

A core testing goal should be to avoid patterns such as:

```rust
sleep(Duration::from_millis(50)); // hope another task ran
```

Tests should instead explicitly drive the scheduler/system until an idle state, event, deadline, or assertion condition.

---

## 17. Whole-userspace and whole-system host execution

A major Linear goal should be to execute nearly complete firmware/application compositions on a host.

Conceptually:

```text
                real application code
                         |
                 real service graph
                         |
                 real libc / std
                         |
                  Linear kernel*
                 /            \
      deterministic scheduler  virtual time
                 |
               fake HAL
                 |
        device models / replay
```

`*` Depending on the test level, a host compatibility backend may replace some kernel machinery; deeper tests may execute the portable kernel core itself.

Desired workflows might eventually resemble:

```text
linear test --component navigation
linear test --graph navigation-replay
linear test --userspace tracker
linear test --system tracker
```

The exact tooling is open; the architectural requirement is that these scopes be possible without redesigning the application.

---

## 18. Configuration concerns

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
| Test bindings | fake/replay/simulated providers |
| Test time policy | real, scaled, manual virtual, auto-advance |
| Fault policy | scripted failures/delays/drops |

A major goal is to avoid recreating Zephyr's Kconfig + DTS + CMake complexity in a different form.

---

## 19. Resource ownership and lifetime

Open requirements:

- exclusive hardware resources should not be instantiated twice;
- shared resources require explicit arbitration/management;
- driver lifetime must be defined for static and optional runtime devices;
- application handles need stable lifetime semantics;
- Rust ownership and a stable C ABI must coexist safely;
- DMA buffers/cache coherency/zero-copy ownership need an explicit model;
- test doubles and simulated devices should obey the same externally visible ownership contracts unless a test deliberately injects a violation.

---

## 20. Initialization and dependency ordering

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

Initialization failure and late availability should be testable through alternative providers and controlled fault injection.

---

## 21. Scheduling and real-time behavior

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

Scheduler policies should be unit-testable independently from architecture-specific context switching, while integration tests validate actual target timing and preemption behavior.

---

## 22. Current design principles

1. **Testability is an architectural property.** Every meaningful dependency boundary should double as a test seam.
2. **Test at arbitrary granularity.** Components, drivers, partial graphs, whole userspace, kernel mechanisms, and full targets should all have appropriate test modes.
3. **Virtualize OS time for tests.** Timers, sleeps, timed waits, and simulated devices should support deterministic and accelerated execution.
4. **Prefer deterministic control over wall-clock sleeps.** Tests explicitly drive time/events/scheduling where practical.
5. **Mechanisms, not concurrency policy.** Kernel supplies threads, queues, synchronization, timers, events, and waiting.
6. **Do not make async part of core service contracts.** Async/Future types stay in opt-in libraries.
7. **Applications depend on capabilities.** Hardware details should rarely appear in normal application code.
8. **Two graphs, not one.** Device composition and logical service composition are distinct.
9. **Services are not schedulable entities by definition.**
10. **Active Objects are a library pattern, not a kernel feature.**
11. **POSIX compatibility is a profile.** Support the useful subset without making Unix semantics internal architecture.
12. **Static by default, dynamic by need.**
13. **Portable core, native escape hatches.**
14. **Host execution is first-class.**
15. **Production and test composition use the same contracts.** Avoid test-only architectural backdoors where normal substitution is sufficient.

---

## 23. Open design questions

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
19. What is the minimum test-double contract each HAL/service interface should support?
20. How should reusable interface conformance test suites be packaged and invoked?
21. Should virtual monotonic time be a build-time backend, a runtime-selectable time domain, or both depending on target/profile?
22. How should `CLOCK_MONOTONIC`, `CLOCK_REALTIME`, and Rust `std::time` map under simulation?
23. What are the deterministic ordering rules for simultaneous virtual timer/IRQ/device events?
24. What are periodic-timer semantics when virtual time jumps over many periods?
25. How much of the kernel can execute as portable host logic versus requiring an architecture/emulator backend?
26. What test scheduler API is required to control/run threads deterministically?
27. Can whole-userspace tests use ordinary Rust `std` threads and sleeps while still being deterministically driven by Linear virtual time?
28. How should real data replay synchronize with virtual time and simulated devices?
29. How should test compositions select arbitrary subgraphs without weakening production dependency validation?
30. How can configuration remain inspectable and significantly simpler than Zephyr's combined machinery?
31. What should the workflow be for adding a board, driver, service provider, AO component, test provider, and application?

---

## 24. Suggested design passes

### A. Kernel mechanisms
Define the minimal thread, synchronization, queue/channel, event, timer, and wait-set primitives.

### B. Time architecture
Define monotonic/realtime time sources, timer queues, hardware compare integration, virtual time, time scaling, auto-advance, and deterministic event ordering before timeout semantics become widespread through APIs.

### C. POSIX / Rust `std` profile
Identify the smallest useful POSIX surface and map libc/Rust `std` requirements onto it, including how `std::time` and timed waits behave under virtual/system tests.

### D. Driver capability model
Define reusable HAL/device interfaces and shared-resource behavior without depending on async, including expectations for fake/model/conformance implementations.

### E. Device/resource resolver
Define board/device description, ownership validation, dependency ordering, product overlays, and test-provider substitution.

### F. Service contracts and composition
Define capabilities, provider selection, lifecycle, language boundaries, and partial-graph construction independently of scheduling.

### G. Active Object prototype
Implement a small optional AO library using only standard OS mechanisms: bounded channels, threads/workers, timers, and run-to-completion dispatch. Verify core AO logic can be tested directly without spawning its production runtime.

### H. Kernel deterministic test harness
Build host-side scheduler, IPC, wait-queue, and timer tests with controlled event/time progression. Avoid real sleeps.

### I. Whole-userspace host test
Run representative real application/service code with fake hardware, virtual time, injected faults, and either host or Linear scheduling primitives.

### J. Optional Embassy prototype
Adapt OS queues/timers/waitables into `Future`s without changing kernel, driver, or service interfaces.

### K. Target conformance
Run common HAL, timing, interrupt, scheduling, and architecture tests on real supported boards/emulators.

---

## 25. Concrete architecture experiments

| Prototype | Question | Success criterion |
|---|---|---|
| POSIX/Rust `std` smoke test | What minimum OS surface is needed? | representative `std::thread`, `std::sync`, `std::net`, `std::fs` subset runs |
| Virtual-time timer test | Can all OS timeout semantics use one controllable monotonic domain? | hours/days of timeout behavior execute deterministically without wall-clock delay |
| Deterministic scheduler test | Can scheduler/IPC/timer interactions be tested without real context switches? | precise, repeatable wake/order assertions; no host sleeps |
| Shared SPI + IMU/Flash | Can ownership, arbitration, DMA and priorities remain composable? | no board-specific application glue; deterministic behavior |
| Simulated SPI/IMU | Can a real driver run unchanged against behavioral fake hardware? | initialization/data/error paths tested entirely on host |
| Service graph resolver | Can `Navigation` be requested independently of physical provider choices? | same application logic builds for two boards and a host |
| Partial graph test | Can one service graph slice be instantiated without whole product startup? | navigation tested against replay providers with normal composition validation |
| Whole-userspace accelerated test | Can ordinary services/threads/timeouts run under virtual time? | complete representative user graph runs significantly faster than realtime |
| AO navigation pipeline | Can reactive components use only threads + bounded channels? | no async in navigation/filter/service core; handler logic tested synchronously |
| Shared AO dispatcher | Can several AOs share workers safely? | lower stack cost while preserving run-to-completion |
| Embassy adapter | Can OS waits/channels/timers become Futures without contaminating core interfaces? | async sample runs; no Embassy dependency below adapter |
| Fault injection | Can bus/network/storage/time failures be scripted at composition boundaries? | reproducible error/recovery tests without production-code hooks |
| HAL conformance suite | Can target HAL implementations share one behavioral suite? | new backend validates against common interface tests |

---

## 26. Non-goals / cautions

- Do not recreate Linux driver complexity without a concrete need.
- Do not pursue full POSIX merely for standards purity.
- Do not make every abstraction dynamically discoverable because POSIX uses runtime handles.
- Do not make Rust async, Embassy, AO, actors, or another runtime mandatory.
- Do not invent a stackful/fiber runtime solely to hide async coloring.
- Do not expose raw hardware resources to ordinary apps merely because the HAL permits it.
- Do not make composition/configuration too magical to inspect.
- Do not make the service graph depend on scheduling policy.
- Do not rely on physical hardware as the primary way to test application/service correctness.
- Do not make tests rely on arbitrary wall-clock sleeps where deterministic time/event control can be provided.
- Do not scatter special test-only code paths through production components when normal interface substitution can provide the required seam.
- Do not assume instruction-level emulation is required for every kernel/system test; use the least expensive simulation fidelity that validates the behavior in question.

---

## 27. Working thesis

> Linear should be a small RTOS that provides conventional threads, synchronization, bounded messaging, timers, waiting, and a useful POSIX compatibility profile for portable C/C++ and Rust `std`; uses capability-oriented abstractions and declarative composition for reusable drivers/hardware; keeps application concurrency policy outside the kernel; and treats deterministic testability as a core architectural requirement rather than an external testing feature.

The architecture should optimize for **simple ordinary code first** while making the same code executable at multiple test scopes:

```text
                          production
                              |
                      application/services
                              |
                  device/service composition
                              |
                            kernel
                              |
                          real HAL

                              ^
                              |
                        same contracts
                              |

                            testing
      +-----------------------+-----------------------+
      |                       |                       |
 component/subgraph      whole userspace        kernel/system
 fake/replay deps       virtual time + HAL    deterministic scheduler
```

The fundamental connection is:

> **Composability and testability are the same mechanism viewed from two directions.** Production composition selects the real providers for a product; test composition selects deterministic, replay, fake, simulated, or fault-injecting providers for the behavior being validated.

For OS-level behavior, Linear should go further than normal dependency injection: the kernel's monotonic time and timer machinery should be virtualizable so normal sleeps, timed waits, poll timeouts, service timers, device models, and substantial portions of scheduling can be controlled and accelerated under test.

This thesis remains provisional until the POSIX/std profile, kernel primitives, virtual-time architecture, deterministic scheduler harness, service contracts, device resolver, partial-graph testing, AO prototype, whole-userspace simulation, and optional async adapter have been measured on representative MCU-class targets.
