# Round 1 — Kubernetes, GitOps, Cloud & Platform Engineering

## 1. Highly available Kubernetes platform for millions of connected vehicles

Do not build one enormous cluster. Vehicle count does not directly determine cluster size; request rate, persistent connections, telemetry volume, service count, workload shape, and failure isolation do.

Use autonomous regional cells:

```text
Vehicles / Mobile Apps
        |
Global traffic management
        |
+----------------+----------------+----------------+
| Region A       | Region B       | Region C       |
| Cell A1/A2     | Cell B1/B2     | Cell C1/C2     |
| Kubernetes     | Kubernetes     | Kubernetes     |
| local queues   | local queues   | local queues   |
| local caches   | local caches   | local caches   |
+----------------+----------------+----------------+
```

Each cell should have multiple availability zones, independent ingress, local queues/caches, bounded vehicle ownership, independent deployment and rollback, and no synchronous cross-region dependency on the normal request path.

Key platform controls:

- Separate node pools for ingress, APIs, stream processing, batch, and system workloads.
- Topology spread constraints, anti-affinity, PodDisruptionBudgets, and priority classes.
- Autoscaling from CPU plus queue lag, concurrent connections, request rate, and processing latency.
- Pre-warmed capacity for predictable reconnect or OTA bursts.
- Default-deny network policy, workload identity, mTLS, signed images, admission policy, and restricted pod security.
- GitOps reconciliation with progressive delivery.
- Regional observability with a federated global view.

Prefer multiple moderate clusters over a few giant clusters. This increases platform management overhead but reduces upgrade blast radius, noisy-neighbor impact, and control-plane pressure.

Example SLOs:

- 99.99% remote-command API availability.
- p99 cloud-side command processing below 500 ms, excluding vehicle wake-up and cellular delay.
- No command executes after expiry.
- Client retries never create duplicate physical effects.

Distinguish command states: accepted, persisted, queued, delivered, validated, executed, acknowledged, expired, or rejected.

## 2. GitOps with Argo CD or Flux

```text
Developer commit
  -> CI tests, lint, SAST, dependency checks
  -> build once
  -> SBOM, provenance, image signing
  -> immutable registry artifact
  -> environment-repo pull request
  -> policy checks and approval
  -> Argo CD reconciliation
  -> canary analysis
  -> promotion or rollback
```

Safeguards:

- Promote immutable image digests; never rebuild per environment.
- Protect production branches with CODEOWNERS and required checks.
- Constrain Argo projects by repository, namespace, cluster, and destination.
- Use narrowly scoped controller credentials.
- Externalize secrets to a secret manager.
- Reject unsigned images, privileged workloads, missing limits, and unapproved registries through admission policy.
- Use sync waves for CRDs, operators, schemas, and applications.
- Use canary or blue/green delivery with automated metric gates.

Drift is classified, not blindly corrected. Some resources are safe to self-heal; others should alert or require review. Emergency mutations must be time-bounded, audited, and reconciled back into Git.

Strong interview line:

> GitOps gives convergence and auditability, but safe delivery still requires artifact integrity, policy enforcement, dependency ordering, progressive exposure, and a tested break-glass path.

## 3. Kubernetes Operator for certificate rotation

First challenge whether a custom operator is needed. Prefer cert-manager or managed PKI unless application-specific lifecycle semantics justify custom code.

A custom controller should:

1. Observe the current certificate and expiration.
2. Acquire a lease to prevent overlapping rotation.
3. Request a certificate using workload identity.
4. Validate chain, SANs, key type, and policy.
5. Store the new key safely.
6. Publish old and new trust during overlap.
7. Trigger controlled reload or rollout.
8. Verify handshakes with synthetic probes.
9. Retire or revoke the previous certificate.
10. Update status, events, and metrics.

Controller properties:

- Idempotent, level-triggered reconciliation.
- Leader election.
- Exponential backoff with jitter.
- Restricted RBAC.
- No private key data in logs or events.
- Conditions for pending, active, degraded, and failed states.
- Metrics for expiry, reconciliation latency, and failed rotation.
- Safe recovery when the issuer succeeds but the controller crashes before updating status.

## 4. Secure hybrid-cloud Kubernetes: AWS and on-premises

Identity:

- Central IdP with short-lived credentials.
- OIDC workload identity in AWS.
- SPIFFE-like workload identity across environments where appropriate.
- No static cloud credentials in pods.
- Just-in-time privilege elevation and audited break glass.

Network:

- Private control-plane endpoints.
- Redundant direct links or VPN.
- Default-deny network policy and explicit egress controls.
- Environment and trust-zone segmentation.
- mTLS based on workload identity, not only IP address.

Supply chain and workload:

- Hardened minimal node images.
- Signed artifacts, attestations, and SBOMs.
- Admission policies and restricted pod security.
- Runtime detection.
- Regular node replacement instead of indefinite patching in place.

Data and operations:

- Envelope encryption and separate key management.
- Regional residency controls.
- Tested backup and restore.
- Unified telemetry schema and policy baseline.
- Network-partition game days.

Each side must remain safe during WAN disconnection; hybrid resilience cannot assume permanent connectivity.

## 5. Prevent Terraform state drift

- Split state by bounded infrastructure domain, account, region, and environment.
- Use encrypted remote state with locking and version history.
- Run plans and applies only through CI.
- Pin providers and module versions.
- Enforce policy for security, cost, naming, and destructive operations.
- Run scheduled drift detection with refresh-only plans.
- Correlate cloud audit events with out-of-band changes.

Classify drift:

1. Legitimate emergency change: import or update code.
2. Unauthorized mutation: revert through Terraform.
3. External resource: import or explicitly exclude.
4. Provider normalization issue: fix module design.
5. Dangerous divergence: freeze deployment until ownership is resolved.

Never blindly apply against unknown drift. Compare configuration, state, actual infrastructure, dependency graph, and proposed replacement/destruction.

Avoid monolithic state, shared credentials, local production applies, casual state editing, and routine use of `-target`.

## 6. Capacity planning

Build a demand model:

```text
Demand = active customers x operations/customer x peak concurrency
         x payload size x dependency amplification
```

Model persistent connections, telemetry events, wake-up bursts, OTA bandwidth, DB writes, queue retention, and regional failover load separately.

Track baseline utilization, peak-to-average ratio, growth, headroom, evacuation capacity, scaling lead time, and cost per unit of customer activity. Validate with load tests, stress tests, saturation tests, downstream-limit tests, and autoscaling-response tests.

A region is not resilient unless surviving regions have enough capacity to absorb failure traffic.

## 7. Terraform, Kubernetes, and Ansible ownership boundaries

- Terraform: cloud resources, IAM, networks, clusters, load balancers, managed data services.
- Kubernetes/GitOps: cluster policy, operators, platform services, and applications.
- Ansible: non-Kubernetes hosts and legacy/on-prem configuration.

Do not allow multiple tools to own the same resource. Prefer immutable images over extensive in-place host configuration.

## 8. Legacy-to-cloud-native migration

Use a strangler strategy rather than a big-bang rewrite:

1. Baseline current behavior, dependencies, SLOs, and traffic.
2. Put a stable API or routing boundary in front of the legacy service.
3. Extract one bounded capability.
4. Add observability and contract tests.
5. Define the data migration path.
6. Shadow traffic and compare output.
7. Shift a small production percentage.
8. Expand progressively with fast rollback.
9. Decommission only after dependency verification.

Data patterns include CDC, backfill plus CDC catch-up, outbox events, and expand-and-contract schema changes. Microservices add network, consistency, deployment, and observability complexity; use them only when independent scaling and ownership justify the cost.
