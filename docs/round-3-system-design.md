# Round 3 — System Design, Scalability & Leadership

## 1. Backend for remote unlock, climate control, and vehicle location

### Requirements and invariants

Clarify registered and connected vehicles, command rate and burst profile, offline duration, TTL, latency expectations, location freshness, privacy, delegated access, and local safety rules.

Core invariants:

- A user can operate only an authorized vehicle.
- A stolen token alone should not be enough for sensitive actions.
- Expired commands never execute.
- Retries do not duplicate a physical effect.
- The vehicle performs final local safety validation.
- Every state transition is auditable.
- Location responses include freshness and accuracy.

### Architecture

```text
Mobile App
   |
Global API Edge / WAF / Rate Limits
   |
Identity + Vehicle Authorization
   |
Command API
   |
Durable Command Store + Transactional Outbox
   |
Regional Message Broker
   |
Vehicle Connection Gateway
   |
Cellular / Wi-Fi
   |
Vehicle Gateway -> local policy -> actuator
   |
Signed acknowledgment
```

Supporting services include device registry, ownership/entitlement, session and key management, vehicle presence, last-known state, audit events, risk evaluation, and notifications.

### Command lifecycle

A request includes vehicle ID, command, idempotency key, creation time, expiry, and nonce. The cloud authenticates the user/device, authorizes access, applies risk controls, persists a signed command envelope, publishes through an outbox, routes to the authoritative vehicle-session cell, and exposes state to the mobile client.

The vehicle validates signature and key version, rejects replay and expiry, checks local safety state, executes idempotently, and returns a signed acknowledgment.

Use a session epoch or fencing token so two regional gateways cannot both act as the authoritative owner.

### Location

Store last-known coordinates, source timestamp, server receive time, accuracy, source, and retention metadata. Never present cached location as live; expose “updated N minutes ago.”

### Security

- Short-lived user tokens and device-bound sessions.
- Step-up authentication for high-risk actions.
- Per-vehicle cryptographic identity protected by secure hardware where available.
- Signed commands, nonce/sequence replay defense, key rotation, and revocation.
- Rate limits by user, device, vehicle, region, and command type.
- Tamper-evident audit logs.
- Separation between telemetry ingestion and command authorization.

Use strong consistency for authorization and command ownership. Presence, telemetry views, and location can generally be eventually consistent.

## 2. Global OTA platform for millions of vehicles

### Safety properties

- Only authorized, compatible software installs.
- Rollout can stop rapidly.
- Downloads resume safely.
- Interrupted or bad installation does not brick the vehicle.
- Every vehicle/artifact relationship is traceable.
- Recovery or rollback is proven before broad rollout.

### Architecture

```text
Source + Hermetic Build
   -> tests, SBOM, provenance
   -> artifact signing
   -> immutable object storage
   -> global CDN

Release Orchestration Control Plane
   -> eligibility and cohort service
   -> signed manifest
   -> vehicle download and verify
   -> A/B installation
   -> health attestation
   -> commit or rollback
```

Separate the control plane, which decides eligibility and rollout policy, from the data plane, which distributes large artifacts through object storage and CDN.

### Progressive rollout

Use internal fleet, hardware-in-the-loop, employee fleet, small canaries, hardware/geography cohorts, percentage waves, and broad deployment. Promotion gates should include download success, verification, install success, boot health, crash regressions, critical service health, and support-event rate.

### Signed manifest

Include artifact digest, compatible hardware, source-version range, target version, size, storage requirements, install preconditions, rollback compatibility, expiry, channel, signature, and key ID.

### Vehicle behavior

- Download by digest using chunks, compression, and resume.
- Verify before installation.
- Install into an A/B or equivalent fail-safe target.
- Atomically switch and validate boot health.
- Roll back automatically on failed health.
- Enforce local preconditions such as parked state, sufficient battery, and thermal limits.
- Retry with bounded exponential backoff and jitter.

### Security

Apply TUF-like principles: offline root, delegated roles, threshold signing for critical releases, key rotation/revocation, anti-rollback controls, immutable artifact paths, provenance, and reproducible/hermetic builds where practical.

Partition blast radius by hardware, source version, geography, carrier, manufacturing batch, and random cohort.

## 3. Driver-preference synchronization across multiple vehicles

Classify preferences by scope and safety:

- Account-level: language and units.
- Driver-level: seat, mirrors, climate.
- Vehicle/model-specific values.
- Mergeable collections such as favorites.
- Non-mergeable selections.
- Safety-sensitive values that cannot apply while driving.

A record should include driver ID, key, scope, value, version, server update time, origin vehicle, and schema version. Partition primarily by driver ID.

Consistency policy:

- Eventual consistency for ordinary UI preferences.
- Stronger guarantees for access-sharing and security-sensitive settings.
- Server-assigned versions or hybrid logical clocks rather than trusting vehicle wall clocks.
- Per-field merge or CRDT only where semantics are clear.
- Explicit conflict handling for destructive changes.

Vehicles keep a durable local cache and mutation log. On reconnect, they submit idempotent mutations with a base version; the server validates, resolves conflicts, returns an authoritative version, and publishes versioned invalidation events.

The cloud synchronizes intent, but the vehicle decides when application is safe. A seat preference must not unexpectedly move the seat at highway speed.

## 4. Monitoring and observability without alert fatigue

### Pipeline

```text
Cloud services / infrastructure / vehicles
   -> collectors and regional buffering
   -> metrics, logs, traces, events
   -> enrichment with topology, firmware, cohort, region
   -> SLO engine and streaming analysis
   -> correlation, deduplication, routing
```

Separate cloud platform telemetry, vehicle fleet telemetry, product SLIs, security signals, and OTA health.

Page only when immediate human action is required, customer SLOs are threatened, automation is insufficient, and a clear owner/runbook exists.

Use:

- Multi-window, multi-burn-rate SLO alerts.
- Symptom alerts before low-level cause alerts.
- Grouping, deduplication, inhibition, and dependency-aware suppression.
- Maintenance windows and cohort/region correlation.
- Tickets rather than pages for nonurgent individual vehicle problems.

Aggregate vehicle failures by firmware, hardware, geography, carrier, cohort, and failure code. One failed vehicle is usually a support event; a statistically meaningful pattern is an SRE incident.

Review alert quality: actionability, duplication, time to acknowledge, time to mitigate, and whether automation could replace the page.

## 5. Multi-region failover

Do not use one strategy for every data class.

Stateless services can run active-active behind global traffic management. Normal request paths should avoid synchronous cross-region calls. Stateful strategy depends on consistency: global strong consistency for critical metadata, authoritative single-writer patterns for ordered workflows, and eventual consistency for caches, presence, and derived telemetry.

Failover requires:

- Capacity in surviving regions.
- Measured replication lag and explicit RPO/RTO.
- Bounded retries with jitter and idempotency.
- Queue buffering and circuit breakers.
- Recoverable session affinity.
- Session epochs or fencing tokens for command ownership.
- Dependency-level health and routing, not only DNS.

Test zone failure, cell failure, regional failure, database promotion, network partition, queue replay, and client retry behavior.

> Failover is not merely a routing feature. It is a capacity, data-consistency, dependency, client-behavior, and operational-readiness feature.

## 6. Millions of real-time telemetry events per second

Start with arithmetic. Two million 500-byte events per second is roughly 1 GB/s raw before protocol overhead, replication, indexing, and multiple consumers.

```text
Vehicles
  -> regional gateways
  -> authentication, schema validation, quotas
  -> partitioned durable event log
  -> stream processors
       -> critical health signals
       -> operational metrics
       -> latest vehicle state
       -> anomaly detection
       -> object storage / analytics
```

Ingestion controls:

- Regional entry points.
- Compact schemas, batching, and compression.
- Schema registry and compatibility rules.
- Device quotas, backpressure, and durable buffering.
- Priority-based load shedding under overload.

Partition by vehicle ID when per-vehicle order matters; include region or event domain where needed. Avoid hot fleet-wide keys. Global ordering is unnecessary; per-vehicle or per-stream ordering is normally sufficient.

Use at-least-once transport with idempotent consumers and event IDs. “Exactly once” has bounded meanings and does not remove the need for idempotency around external side effects.

Stream processing should support event time, watermarks, checkpointing, replay, independent consumer groups, and quarantine streams for malformed events.

Storage tiers:

- Hot: latest state and operational windows.
- Warm: time-series or columnar query store.
- Cold: compressed object storage with lifecycle policies.

Do not index every raw event into an expensive search engine.

Scale consumers from lag and processing time, not just CPU. Plan for reconnect storms and regional evacuation.

## 7. Architectural influence story

A strong answer includes disagreement, evidence, decision process, and measurable outcome.

Example:

> A large operational workflow was designed as one long-running process. Failure near completion caused extensive rework and uncertain state. I proposed treating it as a restartable workflow rather than a script: bounded work units, per-database checkpoints, deterministic transformations, preflight capacity checks, controlled concurrency, and validation at each boundary.
>
> The initial objection was added implementation time. I used observed throughput, failure probability, and restart cost to show that expected completion time must include recovery, not only the happy path. The revised design resumed from the last verified boundary and made operational recovery predictable.
>
> The architectural lesson was that checkpointing, idempotency, and bounded units are both reliability and scalability mechanisms.

Replace generic details with defensible real metrics.

## 8. Balance velocity, reliability, security, and operational excellence

Use explicit risk management rather than opinion:

- Product teams own service outcomes.
- Platform teams provide paved roads.
- Security is automated in CI, artifact verification, and admission.
- Reliability is governed through SLOs and error budgets.
- Higher-risk changes receive stronger controls.
- Repetitive operations become automation.

When a service has budget, teams release through progressive delivery. When burn rate threatens the budget, reduce change exposure and prioritize top reliability risks.

Risk-tier examples:

- Documentation: lightweight checks.
- Stateless internal service: automated canary.
- Authentication or authorization: stronger review and rollback proof.
- Vehicle-command path or OTA signing: separation of duties, staged rollout, cryptographic controls, and explicit recovery validation.

A paved road should include templates, standard telemetry, secrets integration, policy, canary delivery, rollback, SLO dashboards, and cost visibility.

> Velocity is not deployments per day. It is the sustainable rate at which customer value can be delivered without accumulating unacceptable operational risk.
