# linear

**linear** is an experimental embedded RTOS architecture focused on simple, reusable application code across MCU targets.

The current design direction combines:

- portable C/C++ and Rust `std` applications,
- an embedded POSIX compatibility profile rather than POSIX-driven internals,
- capability-oriented HAL and reusable drivers,
- declarative device and service composition,
- concurrency-neutral kernel primitives,
- optional application models such as ordinary threads, Active Objects/state machines, or Rust async runtimes such as Embassy.

The design is intentionally exploratory and is being developed before implementation details are frozen.

## Design

See [docs/architecture.md](docs/architecture.md) for the current architecture plan and open design questions.
