# Interview Playbook — Follow-ups, Red Flags, and Answer Discipline

## What the board is testing

The questions are not primarily testing whether you know product names. They test whether you can:

- Convert ambiguity into explicit requirements.
- Define safety and correctness invariants.
- Design for intermittent connectivity and partial failure.
- Bound blast radius through cells, cohorts, quotas, and progressive delivery.
- Use consistency intentionally rather than ideologically.
- Debug from customer symptom to dependency mechanism.
- Lead incidents under uncertainty.
- Explain trade-offs without hand-waving.

## Recommended answer order

Use this sequence:

1. Clarify the customer operation.
2. Quantify scale, latency, availability, RTO, and RPO.
3. Define non-negotiable invariants.
4. Sketch the request/data flow.
5. Partition by region, tenant, vehicle, or workload.
6. Define consistency and ordering requirements.
7. Explain overload, retries, idempotency, and backpressure.
8. Explain failure modes and recovery.
9. Add security, audit, and privacy.
10. Define SLIs/SLOs and rollout strategy.
11. State trade-offs and evolution path.

Do not start with “I would use Kubernetes, Kafka, Redis, and DynamoDB.” Technology choices should follow requirements.

## Likely follow-up challenges

### Why Kubernetes?

> Kubernetes is an implementation option for the stateless and stream-processing compute layers. It is not the architecture itself, and I would not place every stateful component in it by default.

### What if the vehicle is offline?

> The command has a bounded expiry. Only commands whose semantics permit delay may remain queued. A sensitive unlock command should have a short TTL and must not execute hours later when connectivity returns.

### How do you prevent duplicate unlock?

> The client provides an idempotency key, the cloud persists a unique command ID, and the vehicle remembers recently processed IDs. Redelivery returns the previous result rather than applying the physical action again.

### What if two regions think they own the same vehicle connection?

> The session uses a monotonically increasing epoch or fencing token. The command path and vehicle accept only the newest epoch, preventing a stale regional owner from issuing valid commands.

### Can Kafka guarantee exactly once?

> Kafka can provide transactional guarantees within defined boundaries. Physical effects and external systems still require idempotency. I would state the exact boundary rather than claim universal exactly-once behavior.

### How do you roll back a bad OTA update?

> Use A/B or equivalent fail-safe installation, validate boot health before commit, and revert automatically on failure. Rollback compatibility must be tested because state/schema migrations may make binary rollback unsafe.

### Would you use active-active databases everywhere?

> No. Multi-writer designs introduce conflict and consistency complexity. I use them only where conflict semantics are explicit. Critical ordered workflows may be safer with an authoritative writer and controlled failover.

### What does “high availability” mean here?

Define it numerically and by operation:

- API acceptance availability.
- Command delivery success.
- End-to-end execution latency.
- Maximum stale location.
- Regional RTO and RPO.
- Maximum permitted expired or duplicated execution: zero.

## Red-flag answers

Avoid statements such as:

- “Put everything behind a load balancer.”
- “Kafka scales, so use Kafka.”
- “Kubernetes automatically provides HA.”
- “Prometheus and Grafana solve observability.”
- “Exactly once means there can be no duplicate effects.”
- “Active-active solves regional failure.”
- “Terraform automatically rolls back.”
- “Readiness means the app is healthy.”
- “Microservices are inherently more reliable.”
- “Add retries” without budgets, deadlines, jitter, and idempotency.
- “The cloud unlocks the car” without local vehicle validation.
- “Use AI to reduce alert fatigue” before deterministic SLOs, ownership, and correlation exist.

## Incident interview discipline

During debugging scenarios:

- State that you will stabilize and bound impact before deep root-cause analysis.
- Assign incident roles.
- Establish scope and a timeline.
- Freeze risky changes.
- Follow one failed transaction end to end.
- Compare healthy and unhealthy dimensions.
- Mitigate based on evidence.
- Guard against retry storms during recovery.
- Preserve evidence for postmortem.

Avoid random dashboard tours and command dumping without a hypothesis.

## System-design whiteboard discipline

Draw the smallest useful diagram first:

```text
Client -> Edge -> Auth -> Core Service -> Durable State/Queue -> Device/Worker -> Ack
```

Then add:

- Regional cells.
- Partition key.
- Authoritative writer/owner.
- Async boundaries.
- Failure paths.
- SLO measurement points.
- Security/trust boundaries.

Keep the critical path visually obvious.

## Leadership answer discipline

A principal-level behavioral story should demonstrate:

- The architectural disagreement or risk.
- The evidence you collected.
- Alternatives considered.
- How you influenced peers or leadership.
- The decision and trade-off.
- The measurable result.
- The durable mechanism added to prevent recurrence.

Do not present yourself as the lone hero. Show how you improved the system and the organization’s decision-making.

## Final interview positioning

A strong closing theme is:

> I design systems assuming retries, stale state, dependency failures, regional partitions, bad deployments, and human mistakes will occur. Reliability comes from making those conditions bounded, observable, recoverable, and safe—not from assuming they can be eliminated.
