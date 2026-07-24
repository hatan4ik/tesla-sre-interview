# Tesla SRE / DevOps Interview Preparation

Principal-level preparation for system design, Kubernetes/platform engineering, incident response, reliability, scalability, security, and technical leadership.

> This repository is an independent interview study guide. It does not describe or claim knowledge of Tesla's internal architecture.

> **Canonical shared foundations:** Reusable Linux, Kubernetes, networking, cloud, service-mesh, eBPF, Terraform, observability, reliability, and leadership chapters are maintained in [`hatan4ik/staff-sre-platform-engineering-handbook`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook). This repository owns Tesla/connected-vehicle scenarios, safety invariants, interview adapters, and mock interviews. Existing duplicated foundations remain migration sources until coverage parity is verified.

## Contents

- [Round 1 — Kubernetes, GitOps, Cloud & Platform Engineering](docs/round-1-platform-engineering.md)
  - [Chapter 1 — Highly Available Kubernetes for Millions of Connected Vehicles](docs/round-1/01-ha-kubernetes-connected-vehicles.md)
- [Round 2 — Incident Response, Debugging & Reliability](docs/round-2-incident-response.md)
- [Round 3 — System Design, Scalability & Leadership](docs/round-3-system-design.md)
- [Interview Framework, Follow-ups and Red Flags](docs/interview-playbook.md)

## Shared canonical prerequisites

Use these chapters for reusable theory instead of creating another Tesla-specific copy:

- [Linux Internals module](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/linux)
  - [Architecture, boot, PID 1, systemd, and syscalls](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/01-architecture-boot-syscalls.md)
  - [Processes, scheduling, interrupts, cgroup CPU, and load](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/02-processes-scheduler.md)
  - [Memory, page cache, NUMA, reclaim, PSI, and OOM](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/03-memory.md)
  - [VFS, filesystems, block I/O, NVMe, and latency](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/linux/04-storage-io.md)
- [eBPF, Cilium, Hubble, Falco, and Tetragon](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/core/ebpf-security/cilium-hubble-falco-tetragon.md)
- [Consolidated curriculum map](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/blob/main/curriculum-map.md)

The existing `docs/foundations/01-linux-internals/` files are preserved as migration sources. New generic Linux material should be added to the shared handbook. Tesla files should add only connected-vehicle implications, such as command safety, fleet telemetry, OTA behavior, intermittent connectivity, and vehicle-local authority.

## Principal-level answer framework

Use this sequence for every architecture problem:

1. Clarify requirements and scale.
2. Quantify latency, throughput, availability, RTO and RPO.
3. Define safety and correctness invariants.
4. Separate control plane from data plane.
5. Partition into bounded failure domains.
6. Select consistency per data type.
7. Design retries, idempotency, backpressure and failure recovery.
8. Add security, auditability and privacy.
9. Define SLIs, SLOs and operational signals.
10. Explain trade-offs and an incremental delivery path.

## Core connected-vehicle principle

The cloud requests an action; the vehicle remains the final authority. Every command should be authenticated, authorized, signed, time-bounded, replay-protected, idempotent, auditable, and evaluated against local vehicle safety policy.

## Strong 90-second opening

> I will start by defining the customer operation, availability target, latency boundary, consistency requirement, and safety invariants. For connected vehicles I assume intermittent connectivity, duplicate delivery, stale state, and partial regional failure are normal conditions rather than exceptions.
>
> I would partition the platform into autonomous regional cells, separate control and data planes, and avoid synchronous cross-region dependencies on the critical path. Commands would be durably recorded, idempotent, signed, time-bounded, replay-protected, and correlated end to end. The vehicle remains the final authority for local safety checks.
>
> I would choose consistency by data type: strong consistency for authorization and command ownership, and eventual consistency for presence and telemetry-derived views. Delivery would use progressive exposure, measurable SLOs, bounded retries, automatic rollback, and continuous failure testing.