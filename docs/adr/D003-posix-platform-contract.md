# D003 — POSIX/Unix platform contract

**Status:** Accepted

## Decision

Linear adopts a deliberately selected POSIX/Unix-compatible system contract as a first-class platform interface.

The primary reasons are ecosystem leverage and portability:

- make a Rust `std` port substantially easier by reusing as much of Rust's existing Unix platform abstraction as practical;
- make existing C/C++ libraries and applications written against pthreads, clocks, file-descriptor I/O, polling, sockets, and related APIs easier to port;
- preserve source-level similarity between host/Linux test environments and Linear targets.

Linear does **not** pretend to be Linux and does not require full POSIX conformance.

## Architectural shape

```text
                   Linear kernel mechanisms
                      /             \
                     /               \
          POSIX/Unix contract       native typed APIs
                  |                        |
                libc                  Rust/services
               /    \
          Rust std   C/C++
```

POSIX is therefore more than an optional afterthought, but it is not the internal ontology of every Linear subsystem.

Native Rust and service APIs are free to preserve stronger type, ownership, and capability information rather than exposing everything as integer file descriptors, raw pointers, or `ioctl` commands.

## Rust `std`

Linear should be its own Rust target rather than claiming to be Linux.

Conceptually:

```text
Rust std
   |
shared Unix PAL where semantics match
   |
small Linear-specific PAL/libc pieces where required
   |
Linear POSIX-compatible platform contract
```

This should minimize the amount of Linear-specific `std` platform code while avoiding dependence on Linux-specific syscalls or behaviors.

Rust `std` does not have to route every operation through a public C ABI internally. The important requirement is that Linear's native primitives provide semantics compatible enough for the Unix/POSIX-facing runtime layer to be straightforward.

## Initial high-value compatibility surface

Likely priorities include:

```text
threads:
  pthread_create / join
  mutexes
  condition variables
  semaphores
  TLS

time:
  clock_gettime
  nanosleep
  timed waits

I/O:
  open / close where applicable
  read / write
  file-descriptor lifecycle
  poll/select-like readiness

networking:
  BSD sockets when networking is configured

runtime:
  errno
  malloc/free integration
```

Features such as `fork`, process groups, job control, full terminal/session semantics, and full Unix signal behavior are not implied by this decision.

The exact embedded POSIX profile remains D042.

## Memory-safety implications

The POSIX compatibility surface introduces C-style boundaries such as raw pointers and integer descriptors, but it should not weaken the native Linear programming model.

Unsafe conversion and validation should be localized at the compatibility boundary. Native Rust code should retain typed APIs where possible.

Example:

```text
             native Linear socket object
                 /               \
        typed Rust API       POSIX fd/socket API
```

The kernel implementation should not be redesigned around `int fd` and `void *` merely because POSIX exposes those representations.

## Rationale

A custom-only Linear API would require more bespoke ports for both Rust `std` and existing C/C++ ecosystems. A well-chosen Unix/POSIX-compatible contract amortizes one compatibility effort across Rust `std`, libc, networking libraries, middleware, and existing application code.

This follows the useful portability lesson from NuttX while preserving Linear's ability to use modern typed Rust abstractions internally and at native APIs.

## Follow-up decisions

- D008 — memory protection / execution domains
- D009 — syscall boundary
- D010 — kernel object / handle model
- D015 — thread-local storage
- D042 — exact embedded POSIX profile
- D043 — exact Rust `std` platform profile
- D045 — libc strategy
- D047 — VFS/filesystem model
- D048 — networking model
