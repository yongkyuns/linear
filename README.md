# linear

**linear** is an experimental embedded RTOS architecture focused on simple, reusable, testable application code across MCU targets.

The current design direction combines:

- portable C/C++ and Rust `std` applications,
- an embedded POSIX compatibility profile rather than POSIX-driven internals,
- capability-oriented HAL and reusable drivers,
- declarative device and service composition,
- concurrency-neutral kernel primitives,
- optional application models such as ordinary threads, Active Objects/state machines, or Rust async runtimes such as Embassy,
- first-class host simulation, deterministic testing, fault injection, and virtual/accelerated OS time,
- configurable memory-allocation policy that defaults to general-purpose flexibility while preserving a path to deterministic/safety profiles.

The design is intentionally exploratory and is being developed before implementation details are frozen.

## Design documents

- [Architecture](docs/architecture.md) — narrative system architecture, mechanisms, testing model, and open design areas.
- [Design decisions](docs/decisions.md) — review-oriented decision log with status, rationale, accepted choices, rejected directions, and the next unresolved decision.

When reviewing the project, start with **Design decisions** to see what is already settled, then use **Architecture** for the detailed context.
