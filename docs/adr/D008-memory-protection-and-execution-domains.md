# D008 — Memory protection and execution domains

**Status:** Accepted

## Decision

Linear is **flat-first but protection-capable**.

Safe Rust and explicit capability/resource ownership are the primary software isolation mechanisms for trusted in-image components. Linear does not require every ordinary kernel interaction to cross a privilege/syscall boundary merely because some targets can provide an MPU, PMP, or MMU.

Hardware-enforced isolation remains a first-class optional capability for systems that need stronger fault containment, mixed-language integration, third-party or less-trusted components, security boundaries, or safety-oriented freedom from interference.

Linear therefore treats three mechanisms as complementary rather than interchangeable:

1. **Rust language safety** — ownership, borrowing, encapsulation, and safe APIs.
2. **Capability/authority boundaries** — components receive only the resources/services they are intended to use.
3. **Hardware isolation** — MPU/PMP/MMU-backed protection domains when required by product policy.

A typical trusted Rust system may run as one flat image with no syscall transition between application code and kernel mechanisms. A protected product may place selected components into hardware-isolated domains.

## Protection domains

A protection domain is a **composition/deployment unit**, not inherently a thread, service, Active Object, or process.

For example, a product may compose:

```text
trusted domain
  navigation
  motion
  logging

network domain
  TCP/IP
  TLS
  modem integration

third-party domain
  vendor C component
```

In a flat configuration these boundaries may collapse away. In a protected configuration the same composition metadata should, where practical, drive memory permissions, kernel-object/resource authority, and device access.

This avoids forcing one service per MPU region and acknowledges the limited protection-region resources of many MCU-class MPUs.

## Kernel/API boundary

Public APIs must not expose or require knowledge of private kernel representation simply because a flat build could technically access it.

However, Linear does **not** require an emulated syscall dispatcher in flat builds. Where protection is disabled, public operations may compile/link to direct implementation calls. Where protection is enabled, the relevant calls may pass through validated privileged entry points.

The exact syscall and object/handle representation is intentionally left to D009 and D010.

## Relationship to POSIX and Rust `std`

The first-class POSIX/Unix platform contract does not imply a mandatory Unix-style kernel/userspace split.

The same POSIX-visible operation may map to a direct implementation in a flat build and to a validated kernel entry in a protected build. Likewise, Rust `std` support should preserve the same application-level behavior across protection profiles where practical.

Rust safety also does not make hardware isolation obsolete. Unsafe Rust, DMA, MMIO, FFI, vendor C/C++, protocol stacks, and third-party components can justify an independent fault-containment boundary.

## Dynamic isolated applications

Linear should **architecturally allow independently built, dynamically loaded isolated applications/components in the future**, but this is not a v1 implementation requirement.

Initial MCU profiles may remain statically linked. The object, ABI, protection-domain, and loading designs should avoid unnecessarily blocking a future model such as:

```text
Linear kernel/base image
    + isolated_app_A
    + isolated_app_B
```

The exact executable format, loader, lifecycle, ABI stability, update model, and MMU/MPU constraints are deferred to D064 and related ABI/deployment decisions.

## Rationale

A universal protected-userspace model would impose syscall, validation, memory-region, and integration costs even on small all-Rust systems where safe Rust already provides strong memory-safety guarantees. Conversely, relying on Rust alone would not provide fault containment for unsafe code, C/C++ FFI, DMA, or less-trusted components.

Making protection a composition/profile property preserves low overhead and Rust ergonomics for the common flat case while keeping a credible path to medical, safety, security, mixed-language, plugin, and larger MMU-based systems.

## Consequences

- Flat execution is a first-class configuration, not a temporary implementation shortcut.
- MPU/PMP/MMU protection is a first-class optional architecture capability.
- Protection policy belongs to product composition rather than being inferred from service/thread boundaries.
- Capability/resource declarations should be usable as authority information even without hardware isolation.
- Protected builds may add validated privileged transitions without changing ordinary application-facing contracts.
- Dynamic isolated applications are allowed by the architecture but implementation is deferred.
- D009 (syscall boundary), D010 (kernel object/handle model), D011 (capability/authority model), D064 (loading), and D070 (freedom from interference) build on this decision.
