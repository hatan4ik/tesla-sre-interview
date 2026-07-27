# Tesla Track — Canonical Ownership and Adapter Governance

This repository owns Tesla/connected-vehicle interview context. Reusable engineering foundations belong in [`hatan4ik/staff-sre-platform-engineering-handbook`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook).

## Tesla track owns

The following material should remain here because it changes the architecture, safety model, and interview answer:

- remote vehicle-command lifecycle;
- command expiry, idempotency, replay resistance, and acknowledgement;
- vehicle-local authority and safe refusal;
- session ownership and fencing;
- intermittent and delayed connectivity;
- fleet telemetry and high-volume device events;
- OTA targeting, compatibility, staged rollout, anti-rollback, and unreachable devices;
- driver-profile and preference synchronization;
- hardware-generation and regional cohorts;
- delayed or impossible rollback populations;
- vehicle, mobile, fleet, and customer-support incident paths;
- Tesla-shaped system-design, SRE, and leadership drills.

## Canonical handbook owns

The Tesla track must link rather than fork reusable theory for:

| Shared domain | Canonical owner |
|---|---|
| Linux boot, systemd, processes, memory, storage, networking, and debugging | [`core/linux/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/linux) |
| Kubernetes control plane, scheduling, networking, storage, probes, node repair, node images, runtime, and autoscaling | [`core/kubernetes/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/kubernetes) |
| Service discovery, Envoy request paths, mTLS, SDS, DNS capture, and multi-cluster routing | [`core/service-mesh/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/service-mesh) |
| eBPF, Cilium, Hubble, Falco, and Tetragon | [`core/ebpf-security/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/ebpf-security) |
| Workload identity, secrets, supply-chain security, provenance, and artifact trust | [`core/security/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/security) |
| GitOps, progressive delivery, and Terraform governance | [`core/delivery-gitops/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/delivery-gitops) and [`core/infrastructure-as-code/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/infrastructure-as-code) |
| Metrics, logs, traces, profiling, OpenTelemetry, alerting, and evidence | [`core/observability/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/observability) |
| Request-path debugging, cohort analysis, and postmortems | [`core/incident-response/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/incident-response) |
| SLOs, overload, graceful degradation, blast radius, DR, failback, and chaos | [`core/reliability/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/reliability) |
| Consistency, retries, idempotency, queues, streams, caching, replication, and fencing | [`core/distributed-systems/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/distributed-systems) |
| Platform product, policy, tenancy, fleets, portals, and golden paths | [`core/platform-engineering/`](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/core/platform-engineering) |

## Adapter standard

A Tesla interview chapter should contain:

1. the exact connected-vehicle or fleet question;
2. safety, scale, connectivity, and authority assumptions;
3. a concise spoken answer;
4. links to exact canonical prerequisites;
5. vehicle/mobile/fleet-specific state machines and failure modes;
6. safety invariants and local-authority rules;
7. rollout, rollback, and unreachable-population behavior;
8. fleet and customer-facing SLIs;
9. adversarial follow-ups;
10. one truthful personal-story bridge.

It should not repeat generic Linux, Kubernetes, Terraform, service-mesh, observability, SLO, or distributed-systems textbooks.

## Cross-track examples

### Remote unlock

Canonical prerequisites:

- workload/device identity;
- idempotency and command state;
- queues and delivery semantics;
- fencing and leases;
- request-path incident response;
- SLOs and protected cohorts.

Tesla adapter adds:

- authenticated vehicle session;
- command expiry;
- online, sleeping, disconnected, and delayed vehicle states;
- vehicle-local safety refusal;
- mobile acknowledgement versus actual vehicle execution;
- duplicate/replayed command protection;
- customer-support evidence.

### OTA rollout

Canonical prerequisites:

- artifact trust and provenance;
- staged delivery;
- blast-radius engineering;
- overload and telemetry backpressure;
- node/image qualification concepts;
- chaos and recovery validation.

Tesla adapter adds:

- hardware and firmware compatibility;
- battery, bandwidth, and parked-state constraints;
- A/B or equivalent local rollback;
- anti-rollback and safety authority;
- unreachable or offline cohorts;
- fleet abort thresholds;
- delayed recovery evidence.

### Driver-profile synchronization

Canonical prerequisites:

- consistency models;
- versioning and conflict resolution;
- event delivery and idempotency;
- caching and offline behavior;
- multi-region replication.

Tesla adapter adds:

- vehicle-local changes;
- mobile/cloud changes;
- intermittent connectivity;
- preference safety classification;
- driver and vehicle ownership boundaries;
- deterministic merge or conflict rules.

## Executable shared labs

Use the canonical labs for:

- retry amplification and idempotency;
- fencing tokens and stale-writer rejection;
- queue redelivery and transactional outbox;
- overload, tenant priority, and failover headroom;
- disaster-recovery state machines;
- OpenTelemetry pipeline and Collector integration;
- service-mesh identity, DNS, and failover contracts;
- Kubernetes node repair, probes, scheduling, and graceful drain;
- artifact trust, fleet rollout planning, and secret rotation.

[Browse all shared labs](https://github.com/hatan4ik/staff-sre-platform-engineering-handbook/tree/main/labs)

## No-duplication workflow

Before changing or adding a chapter:

1. search the canonical handbook by topic and failure signature;
2. extend the canonical chapter when the knowledge is company-neutral;
3. retain only connected-vehicle, fleet, safety, and organizational context here;
4. link exact prerequisites rather than copying them;
5. mark hypothetical scale and architecture explicitly;
6. never convert a lab or design exercise into a production claim.
