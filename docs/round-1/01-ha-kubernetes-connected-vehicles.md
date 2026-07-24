# Chapter 1 — Highly Available Kubernetes for Millions of Connected Vehicles

## Interview question

> How would you design a highly available Kubernetes platform that supports millions of connected vehicles?

## What the interviewer is actually testing

This is not primarily a Kubernetes trivia question. It tests whether you can turn a vague scale claim into a bounded architecture, separate customer traffic from platform mechanics, define failure domains, make state and consistency decisions, and operate the result safely.

A weak answer lists Kubernetes features. A principal-level answer explains why the platform is partitioned, what remains available during failure, how traffic and state move, what the SLO means, and where Kubernetes stops being the right abstraction.

## Strong opening answer

> I would not translate “millions of vehicles” into one giant Kubernetes cluster. I would first quantify concurrent connections, command rate, telemetry rate, payload size, regional distribution, latency targets, and the percentage of vehicles that are simultaneously online. I would then build autonomous regional cells, each spanning multiple availability zones and containing bounded compute, messaging, caching, and data partitions. Kubernetes would run the stateless APIs, gateways, and selected stream-processing workloads, while globally critical state would use purpose-built managed data systems. The normal request path would not depend synchronously on another region. Capacity, deployment, observability, and failover would all be designed at the cell boundary so a failed cluster, zone, release, or dependency cannot become a global outage.

That opening establishes the architecture before discussing products.

---

## 1. Clarify requirements before designing

Ask a small number of high-value questions.

### Scale

- How many registered vehicles exist?
- What percentage are concurrently connected?
- Are connections persistent, periodic, or wake-on-demand?
- What is the steady-state and peak command rate?
- What is the telemetry event rate and average payload size?
- How large are reconnect storms after carrier or regional outages?

### Latency

Separate cloud processing from cellular and vehicle wake-up time.

- Remote command API acceptance target.
- Command delivery target for an already-connected vehicle.
- Vehicle wake-up target.
- Telemetry ingestion and processing latency.
- Freshness requirement for last-known state and location.

### Availability and durability

- Availability target for command acceptance.
- Maximum tolerable command loss.
- Whether command status must survive regional loss.
- RTO and RPO by data class.
- Required behavior when the vehicle is offline.

### Safety and consistency

- Which actions are safety sensitive?
- Which state requires strong consistency?
- Can commands be executed after reconnect?
- How long should commands remain valid?
- What ordering is required per vehicle?

### Compliance and privacy

- Regional data residency.
- Retention of location and telemetry.
- Separation of personally identifiable information from operational telemetry.
- Audit requirements for vehicle access and administrative actions.

## 2. Example sizing model

State assumptions explicitly and make the math easy to follow.

Assume:

- 10 million registered vehicles.
- 20% concurrently connected at a busy period: 2 million connections.
- 100,000 remote commands per second during a major burst.
- 2 million telemetry events per second.
- Average telemetry event size of 500 bytes before protocol overhead.

Raw telemetry ingress is approximately:

```text
2,000,000 events/s × 500 bytes = 1 GB/s
```

That is before replication, indexing, stream processing, retries, and derived events. With a replication factor of three and multiple consumers, internal throughput can become several gigabytes per second.

The exact numbers are less important than demonstrating that you size connections, commands, telemetry, storage, and failure capacity separately.

## 3. Top-level architecture: global platform, regional cells

```text
                         +--------------------------+
Mobile apps ------------>| Global API edge / WAF   |
                         | traffic steering         |
                         +------------+-------------+
                                      |
             +------------------------+------------------------+
             |                         |                        |
      +------v------+           +------v------+          +------v------+
      | Region A    |           | Region B    |          | Region C    |
      |             |           |             |          |             |
      | Cell A1     |           | Cell B1     |          | Cell C1     |
      | Cell A2     |           | Cell B2     |          | Cell C2     |
      +------+------+           +------+------+          +------+------+
             |                         |                        |
      Regional data             Regional data            Regional data
      and event log             and event log            and event log

Vehicles connect to the nearest healthy regional vehicle gateway.
Global control metadata is replicated using consistency appropriate to each data class.
```

### Why cells instead of a giant global cluster?

A cell is a bounded deployment and failure domain. It owns a known subset of traffic, vehicles, queues, caches, and service instances.

Cells reduce:

- Upgrade blast radius.
- Control-plane pressure.
- Noisy-neighbor impact.
- Risk from malformed traffic.
- Impact of configuration mistakes.
- Difficulty of capacity attribution.
- Complexity of regional evacuation.

Cells add operational overhead, so the platform team must automate cluster creation, policy, observability, upgrades, and application placement.

A useful design target is that losing one cell should degrade only that cell's assigned population, and traffic should be drainable or rehomed without changing the entire platform.

## 4. Workload placement

Kubernetes is appropriate for:

- Stateless mobile APIs.
- Authentication and authorization adapters.
- Command orchestration services.
- Vehicle connection gateways, if connection behavior and kernel tuning are well understood.
- Stream-processing workers.
- Internal control-plane services.
- Observability collectors.

Kubernetes is not automatically the best place for:

- Globally consistent databases.
- Large durable event logs when a managed broker provides stronger operational guarantees.
- Object storage.
- Cryptographic root-key systems.
- Workloads requiring specialized hardware or strict isolation that is easier outside the general cluster.

Strong interview line:

> Kubernetes is the compute substrate, not the entire architecture.

## 5. Cluster and node-pool design

Each cell should use a cluster or small set of clusters spanning at least three availability zones where the cloud and workload permit it.

Separate node pools by workload characteristics:

### System pool

Runs DNS, networking components, policy engines, metrics agents, and core controllers. It should have protected capacity and strict admission controls.

### Edge and connection gateway pool

Optimized for high network throughput, large connection counts, predictable kernel settings, and careful disruption management.

### Stateless API pool

General application workloads with horizontal autoscaling.

### Stream-processing pool

Memory and CPU profiles tuned for event processing, checkpointing, and broker locality.

### Batch and maintenance pool

Lower-priority jobs that can be preempted or paused during incidents.

### Sensitive workload pool

Dedicated nodes, stronger isolation, taints and tolerations, and possibly confidential-compute capabilities where justified.

Use taints, tolerations, affinity, topology spread constraints, and quotas to keep workload classes isolated.

## 6. Scheduling and availability controls

### Topology spread constraints

Spread replicas across zones and nodes. Anti-affinity alone can be too rigid at large scale, while topology spread gives more controlled skew.

### PodDisruptionBudgets

Use PDBs to protect against voluntary disruption, but avoid values that make node upgrades impossible. A PDB does not protect against node loss or application failure.

### Priority classes

Reserve high priority for ingress, command-path services, DNS, networking, and core controllers. Do not let every team classify its service as critical.

### Requests and limits

Requests are required for sensible scheduling and capacity planning. Limits must be selected carefully:

- CPU limits can cause CFS throttling and latency even when nodes look healthy.
- Memory limits provide isolation but can trigger OOM termination.
- Critical latency-sensitive services may use guaranteed QoS where justified.

### Graceful termination

Connection-oriented services must:

1. Stop accepting new connections.
2. Mark themselves unready.
3. Drain or transfer existing sessions.
4. Persist required session state.
5. Exit before the termination grace period.

A rolling deployment that drops hundreds of thousands of vehicle connections can create a reconnect storm larger than the original deployment traffic.

## 7. Vehicle connection ownership

A connected vehicle should have one authoritative session owner at a time.

The platform can maintain:

```text
vehicle_id -> region -> cell -> gateway -> session_epoch
```

Use a lease, session epoch, or fencing token so stale gateways cannot continue issuing valid commands after ownership changes.

Example:

1. Gateway A owns vehicle V at epoch 104.
2. Connectivity moves and Gateway B acquires epoch 105.
3. Any command or acknowledgment from epoch 104 is rejected as stale.

This avoids split-brain connection ownership during network partitions or failover.

Presence state can be eventually consistent for discovery, but command routing must validate authoritative session ownership before delivery.

## 8. Autoscaling strategy

CPU alone is insufficient.

Scale on workload-specific signals:

### API services

- Requests per second.
- In-flight requests.
- p95 or p99 latency.
- Connection-pool saturation.
- CPU and memory.

### Vehicle gateways

- Active connections per pod.
- New connections per second.
- Network throughput.
- File descriptor usage.
- Event-loop lag.
- TLS handshake rate.

### Stream processors

- Broker partition lag.
- Oldest unprocessed event age.
- Processing time per event.
- Checkpoint duration.
- CPU and memory.

### Node capacity

Use Cluster Autoscaler, Karpenter, or an equivalent system. Account for node startup time and cloud quota. For predictable OTA or reconnect events, pre-scale instead of relying exclusively on reactive provisioning.

### Scale-down protection

Avoid aggressive scale-down of connection gateways and stateful processors. Draining may be expensive and can trigger reconnection or repartitioning storms.

## 9. Capacity for failure

Normal utilization is not enough. Plan for:

- Loss of one availability zone.
- Loss of a cell.
- Regional traffic evacuation.
- Deployment rollback overlap.
- Reconnect storms.
- OTA campaign bursts.
- Broker replay after downstream recovery.

A simple regional capacity rule might be:

```text
usable capacity after one-zone loss >= projected peak demand + safety margin
```

For regional failover, either reserve evacuation capacity in neighboring regions or define degraded modes. A platform is not multi-region merely because identical clusters exist in multiple regions.

## 10. Data architecture and consistency

Classify data instead of choosing one database for everything.

### Strong consistency candidates

- Vehicle ownership and delegated access.
- Command idempotency records.
- Command authorization decisions.
- Active session epoch or fencing token.
- OTA release signing and promotion metadata.

### Eventual consistency candidates

- Presence views.
- Last-known telemetry.
- Aggregated fleet health.
- Cached driver preferences.
- Search indexes.

### Durable event log

Use a partitioned event system for telemetry and command state transitions. Partition by `vehicle_id` when per-vehicle ordering matters.

Do not claim global ordering. It is rarely required and is expensive. Per-vehicle or per-workflow ordering is usually sufficient.

### Idempotency

Use an immutable command ID or client idempotency key. Retries must return the existing command result rather than generate another physical action.

### Outbox pattern

When the command API must update a database and publish to a broker, use a transactional outbox or equivalent to avoid a dual-write gap.

## 11. Network architecture

### Ingress

Separate mobile API traffic from vehicle connectivity. Their protocols, connection duration, rate patterns, security controls, and scaling behavior differ.

### Egress

Use explicit egress policy and predictable NAT capacity. Large fleets can exhaust ports or create unexpected cross-zone traffic costs if egress is not modeled.

### Service-to-service communication

- Use workload identity.
- Encrypt sensitive paths.
- Apply timeouts, bounded retries, and circuit breakers.
- Avoid synchronous cross-region calls on the critical path.
- Prefer local dependencies within the cell.

### DNS

Monitor DNS latency, errors, cache behavior, and query volume. DNS failures frequently appear as generic application latency.

### Service mesh

A mesh may help with identity, encryption, routing, and telemetry, but it adds resource cost and another failure layer. At high scale, restrict configuration scope so every proxy does not receive a global service registry.

## 12. Security architecture

### Identity

- Short-lived workload credentials.
- No static cloud keys in pods.
- Separate human, workload, and automation identities.
- Just-in-time administrative access.

### Supply chain

- Hermetic or controlled builds.
- SBOM generation.
- Artifact signing and provenance.
- Admission verification.
- Immutable deployment by digest.

### Runtime

- Default-deny network policy.
- Restricted pod security.
- Read-only root filesystem where practical.
- Minimal capabilities.
- Seccomp or equivalent runtime profiles.
- Dedicated nodes for high-trust workloads.

### Secrets and keys

- External secret manager.
- Envelope encryption.
- Rotation and revocation.
- No sensitive material in logs, environment dumps, or Kubernetes events.

### Vehicle trust boundary

The cloud does not directly force a physical action. It sends a signed, authorized, expiring command. The vehicle validates the command and applies local safety policy.

## 13. Deployment and change safety

Use GitOps and progressive delivery at the cell boundary.

A safe release flow:

1. Build once and sign the artifact.
2. Deploy to test and staging.
3. Deploy to one low-risk canary cell.
4. Compare latency, error rate, resource usage, connection churn, and command success.
5. Expand to a small percentage of production cells.
6. Promote region by region.
7. Stop automatically if guardrails fail.

For connection gateways, include:

- Connection drop rate.
- Reconnect rate.
- TLS handshake failures.
- Session migration success.
- Commands delayed during rollout.

Configuration changes deserve the same controls as code. Many severe outages come from valid but unsafe configuration.

## 14. Observability

Instrument the customer journey, not only Kubernetes.

### Remote-command lifecycle

```text
request_received
-> authenticated
-> authorized
-> persisted
-> published
-> routed_to_gateway
-> delivered_to_vehicle
-> accepted_or_rejected
-> executed
-> acknowledged
```

Every transition should carry a correlation identifier.

### Platform signals

Use RED for request-driven services:

- Rate.
- Errors.
- Duration.

Use USE for resources:

- Utilization.
- Saturation.
- Errors.

Also monitor:

- Active vehicle connections.
- Reconnect rate.
- Broker lag and oldest-event age.
- Command expiry rate.
- Duplicate suppression rate.
- Cross-zone and cross-region traffic.
- DNS and TLS latency.
- API-server and etcd health.
- Pending pods and node-provisioning latency.

### Avoid high-cardinality mistakes

Do not put raw `vehicle_id` into general Prometheus labels. Use traces, logs, or purpose-built event stores for per-vehicle investigation.

## 15. SLOs and error budgets

Define SLOs around customer outcomes.

Example command-path SLIs:

- API availability.
- Successful command persistence.
- Command delivery success for connected vehicles.
- End-to-end acknowledgment latency.
- Expired command rate.
- Incorrect duplicate execution count.

Example targets:

- 99.99% command API availability.
- p99 cloud processing below 500 ms, excluding cellular delay and wake-up.
- 99.9% of commands for already-connected vehicles delivered within a defined threshold.
- Zero execution of cryptographically invalid or expired commands.
- Zero duplicate physical effects for the same command ID.

The last two are correctness invariants, not merely availability percentages.

Use multi-window, multi-burn-rate alerts so operators are paged on meaningful error-budget consumption rather than transient metric spikes.

## 16. Failure scenarios and expected behavior

### One pod fails

Traffic shifts to healthy replicas. The session owner changes only if required. Retry behavior is bounded and idempotent.

### One node fails

Pods reschedule across zones. Connection services reconnect gradually with jitter and admission controls.

### One availability zone fails

Remaining zones have reserved capacity. Zonal dependencies are isolated. Traffic steering avoids the failed zone.

### One cell fails

The cell stops receiving new traffic. Assigned vehicles reconnect to another cell or degraded service is declared. Durable command state remains recoverable.

### One region fails

Global routing moves mobile API traffic. Vehicle sessions reconnect to designated regions. Data failover follows the consistency model. Neighboring regions have enough capacity or activate a documented degraded mode.

### Broker unavailable

The command API persists accepted commands but does not falsely claim delivery. Outbox publication resumes after recovery. Backpressure and admission control prevent unbounded buildup.

### Database unavailable

Authorization-sensitive commands fail closed. Read-only or stale-safe features may continue from cache where policy permits.

### Control plane unavailable

Existing pods and data-plane traffic continue. No assumption is made that scheduling or deployment can proceed until the control plane recovers.

### Bad deployment

Canary metrics stop promotion. Rollback occurs at the cell level. The system avoids replacing all healthy replicas simultaneously.

## 17. Disaster recovery

For every stateful component, specify:

- RPO.
- RTO.
- Replication method.
- Backup frequency.
- Restore procedure.
- Validation method.
- Ownership.

Backups are not a recovery strategy until restores are tested.

Run game days for:

- Zone loss.
- Cell evacuation.
- Region evacuation.
- Broker replay.
- Database promotion.
- Secret and certificate failure.
- DNS failure.
- Cloud quota exhaustion.

## 18. Cost and efficiency

Principal engineers include cost in the architecture.

Track unit economics such as:

- Cost per thousand active vehicle connections.
- Cost per million commands.
- Cost per billion telemetry events.
- Cross-zone traffic cost.
- Storage cost per retention tier.

Optimization examples:

- Batch and compress telemetry.
- Keep processing local to the region.
- Use tiered storage.
- Avoid indexing all raw telemetry into expensive search systems.
- Right-size sidecars and platform agents.
- Use spot capacity only for interruptible workloads.

Do not trade away regional resilience or command safety for small compute savings.

## 19. Trade-offs to state explicitly

### More clusters versus fewer clusters

More clusters reduce blast radius and control-plane scale risk but increase automation and operational overhead.

### Active-active versus active-passive data

Active-active improves locality and availability but requires explicit conflict resolution. Critical ordered workflows may be safer with a single authoritative writer and controlled failover.

### Service mesh versus simpler networking

A mesh standardizes identity and telemetry but adds latency, resource overhead, and another control plane.

### Managed services versus self-operated systems

Managed services reduce operational toil but can create provider dependency and quota constraints. The decision should follow reliability requirements and organizational competence.

### Higher utilization versus failure headroom

Running near saturation improves apparent efficiency but causes nonlinear latency and leaves no capacity for failure or traffic bursts.

## 20. Common weak answers

Avoid:

- “Use a multi-master Kubernetes cluster across the world.”
- “Kubernetes automatically provides high availability.”
- “Use Kafka because Kafka scales.”
- “Put a load balancer in front.”
- “Autoscale on CPU.”
- “Use active-active databases everywhere.”
- “Exactly-once delivery prevents duplicates.”
- “Prometheus and Grafana provide observability.”
- “Deploy across three zones” without discussing zonal dependencies and capacity.
- “Fail over to another region” without explaining data, ownership, capacity, and client behavior.

## 21. Follow-up questions and model responses

### Why not one very large cluster?

> A very large cluster can be technically possible, but it increases control-plane contention, upgrade blast radius, policy complexity, and the impact of a bad deployment or noisy workload. Cells let us scale horizontally while keeping failure and ownership boundaries understandable.

### How many clusters would you use?

> I would not pick a number from vehicle count alone. I would size cells from connection count, workload limits, failure budget, deployment risk, cloud quotas, and operational cost. I would establish a tested maximum cell size and add cells before approaching it.

### Would you run Kafka inside Kubernetes?

> It depends on the organization’s operational maturity and the managed options available. Kafka can run successfully on Kubernetes, but for a critical global event backbone I would compare failure recovery, storage behavior, upgrade risk, and staffing against a managed service. Kubernetes is not a goal by itself.

### How do you avoid duplicate unlock commands?

> Use a durable command ID and idempotency record in the cloud, at-least-once delivery, and duplicate suppression in the vehicle. The physical side effect is protected at the final executor; broker-level exactly-once claims are not enough.

### What happens when the vehicle is offline?

> The API reports accepted or queued, not executed. Every command has a type-specific TTL. A sensitive command such as unlock should expire quickly and must not execute hours later after connectivity returns.

### What if two gateways believe they own the same vehicle?

> Session ownership uses a monotonically increasing epoch or fencing token. The vehicle and command path accept only the newest epoch, so the stale gateway cannot continue issuing valid actions.

### How do you deploy without disconnecting the fleet?

> Use small cell-level canaries, connection draining, surge capacity, long enough termination windows, and rollout gates based on disconnect and reconnect metrics. For high-connection services, deployment safety is a session-management problem, not only a Deployment strategy.

### What is the most important alert?

> A customer-outcome alert such as elevated command failure or excessive end-to-end latency, segmented by region and cell. Infrastructure alerts should provide diagnosis, but the page should begin with customer impact.

## 22. Whiteboard answer structure

Draw in this order:

1. Clients: mobile applications and vehicles as separate entry paths.
2. Global traffic management.
3. Three regions.
4. Two cells per region.
5. Kubernetes compute in each cell.
6. Regional broker and data systems.
7. Global metadata only where required.
8. Command and telemetry paths.
9. Failure boundaries.
10. SLO and observability points.

Do not draw every microservice. Keep the first diagram at the system level, then zoom into command routing or vehicle sessions when asked.

## 23. Five-minute spoken answer

> I would begin by separating registered vehicles from concurrent connections and by quantifying command rate, telemetry throughput, latency, RTO, RPO, and regional distribution. I would not build a single massive Kubernetes cluster. I would build a global platform from autonomous regional cells. Each cell would span multiple availability zones and contain its own ingress, stateless services, vehicle gateways, queues, caches, and bounded data partitions. The normal request path would remain inside the region and ideally inside the cell.
>
> Kubernetes would be the compute substrate for APIs, gateways, and selected stream processors, but I would use purpose-built systems for durable event logs, globally consistent metadata, object storage, and high-trust signing systems. Workloads would be separated into node pools, spread across zones, protected with disruption budgets and priority classes, and autoscaled using workload signals such as connection count, request concurrency, and broker lag rather than CPU alone.
>
> A vehicle connection would have one authoritative owner identified by a session epoch or fencing token. Commands would be persisted before publication, delivered at least once, and made safe through end-to-end idempotency and expiry. Strong consistency would be used for ownership, authorization, and command identity, while presence and telemetry-derived views could be eventually consistent.
>
> Availability would be measured through customer SLOs such as command API success, delivery latency for connected vehicles, command expiry, and duplicate suppression. Deployments would proceed cell by cell with canary gates based on error rate, latency, connection churn, and command success. Capacity would include zone-loss and regional evacuation headroom, because multiple regions without spare capacity are not meaningful failover.
>
> Finally, I would test the design continuously through zone, cell, broker, database, and regional failure exercises. The objective is not merely that Kubernetes remains healthy; it is that customer operations remain safe, observable, and recoverable under partial failure.

## 24. One-minute compressed answer

> I would use autonomous multi-AZ regional cells rather than a giant global Kubernetes cluster. Each cell would own a bounded set of vehicle connections and contain local ingress, APIs, gateways, queues, caches, and observability, with no synchronous cross-region dependency on the normal command path. Kubernetes would run compute, while durable brokers and critical databases would use purpose-built systems. I would autoscale using connection count, request concurrency, and queue lag; protect connection workloads with graceful draining and surge capacity; and use session epochs to prevent split-brain vehicle ownership. Strong consistency would apply to authorization, command identity, and session fencing, while presence and telemetry views could be eventually consistent. Releases would be canaried by cell, and SLOs would measure customer command outcomes rather than cluster health. Capacity and disaster recovery would be tested for zone, cell, and regional loss.

## 25. Final principal-level takeaway

The strongest answer is not “Kubernetes at scale.” It is:

> A globally distributed connected-vehicle platform composed of bounded regional cells, where Kubernetes is one replaceable compute layer, command safety is enforced end to end, state has explicit consistency semantics, and every failure is contained, observable, and recoverable.