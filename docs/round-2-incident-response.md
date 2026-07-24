# Round 2 — Incident Response, Debugging & Reliability

## 1. Mobile users suddenly cannot unlock vehicles

Restore safe service first; root cause comes second.

Immediate actions:

1. Declare the incident.
2. Assign incident commander, operations lead, communications lead, and SMEs.
3. Establish scope by region, app version, vehicle firmware, account type, and failure stage.
4. Freeze risky changes and inspect recent deployments/configuration.
5. Trace the command path end to end.

```text
Mobile app
 -> API edge
 -> authentication
 -> authorization / ownership
 -> command service
 -> durable store / broker
 -> vehicle connection gateway
 -> cellular or Wi-Fi network
 -> vehicle validation
 -> actuator execution
 -> acknowledgment
```

Measure success and latency at every transition: API response codes, auth failures, authorization denials, enqueue success, broker lag, active connections, delivery, vehicle acknowledgment, execution, and rejection reason.

An HTTP 200 means only that the cloud accepted the request unless the API explicitly waits for execution. Report precise states: accepted, queued, delivered, rejected, expired, executed.

Mitigation options depend on evidence:

- Roll back a recent release.
- Disable a feature flag.
- Route away from an unhealthy cell.
- Bypass a nonessential dependency.
- Scale consumers only when lag is proven.
- Rate-limit recovery traffic and client retries.
- Preserve alternate physical-access guidance in customer communication.

After stabilization, preserve evidence, quantify impact, identify containment gaps, and add end-to-end synthetic command tests using test vehicles or simulators.

## 2. Nodes healthy but API latency doubles

Node health is not service health.

Start by defining the symptom:

- p50, p95, or p99?
- All routes or one endpoint?
- One region, zone, version, tenant, or dependency?
- Did traffic volume, payload shape, or error rate change?
- Is latency observed at the client, edge, application, or dependency?

Decompose request latency with traces:

```text
edge + queueing + application + database + downstream + serialization + network
```

Inspect:

- CPU throttling, not just utilization.
- Memory pressure and garbage collection.
- Thread and connection pools.
- DNS and TLS latency.
- Storage latency.
- DB locks and query plans.
- Cache hit-rate collapse.
- Broker lag.
- Service-mesh proxy overhead.
- Cross-zone routing.
- Endpoint imbalance.
- Retry amplification.

Compare a healthy request/pod with an unhealthy one rather than relying on fleet averages.

## 3. OTA rollout fails for a subset of vehicles

Classify the subset before choosing a theory:

- Vehicle model and hardware revision.
- Current and target firmware.
- Geography, carrier, ASN, and CDN edge.
- Battery, charging state, and storage availability.
- Bootloader version.
- Rollout cohort.
- Exact failure stage.

The rollout state machine should expose:

```text
eligible -> manifest received -> download started -> download complete
-> signature verified -> preconditions passed -> install started
-> rebooted -> health verified -> committed
```

Infrastructure indicators include regional service errors, queue lag, object-store errors, and failures independent of hardware. Network indicators include carrier/ASN or CDN concentration, timeout, packet loss, DNS, TLS, or partial downloads. Application/firmware indicators include hardware-specific incompatibility, signature rejection, install failure after successful download, and boot-health failure.

Pause the affected cohort, preserve healthy cohorts, stop promotion, validate rollback safety, capture failure artifacts, and avoid retry storms.

## 4. Prometheus shows errors but dashboards hide root cause

Dashboards are curated summaries. Move from aggregation to evidence:

1. Identify the exact metric, labels, and time window.
2. Slice by service, route, zone, version, status code, and customer class.
3. Follow a metric exemplar to a trace.
4. Inspect correlated structured logs.
5. Compare healthy and failing requests.
6. Correlate with deployment and dependency changes.

Also validate the metric itself: counter resets, scrape gaps, wrong denominator, double instrumentation, hidden retries, or lost label dimensions.

> The dashboard tells me where to start. A representative failed request and its dependency path reveal the failure mechanism.

## 5. Terraform partially fails

Do not immediately rerun `apply`.

Recovery process:

1. Preserve failed-run output and plan.
2. Verify or acquire the state lock.
3. Compare configuration, state, and actual provider resources.
4. Refresh and inspect the proposed plan.
5. Identify asynchronous or partial side effects.
6. Decide per resource: retry, import, replace, remove stale state, or revert manually and reconcile.
7. Review a recovery plan.
8. Apply carefully and validate externally.

Be especially cautious with IAM, DNS, networking, databases, encryption keys, and stateful systems.

Long-term prevention: smaller state domains, preflight validation, provider timeouts, safer module contracts, state versioning, post-deployment verification, and failure-injection testing.

## 6. Pods continuously restart while readiness is healthy

Readiness does not explain restarts. Inspect termination state:

```bash
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl logs <pod> --previous
kubectl get pod <pod> -o json
kubectl get events --sort-by=.lastTimestamp
```

Check `restartCount`, `lastState.terminated`, exit code, signal, container/sidecar state, init containers, OOMKilled, liveness/startup probe failures, evictions, deployment revisions, lifecycle hooks, and node events.

Common mechanisms:

- The process becomes ready, leaks memory, and is OOM-killed.
- Liveness is too aggressive while readiness is usually successful.
- A sidecar exits and pod restart behavior restarts containers.
- The workload mishandles SIGTERM.
- A controller or rollout repeatedly replaces the pod.

Mitigate by rollback, correcting limits, fixing the leak, adding startup probes, removing dependency checks from liveness, fixing signal handling, or correcting sidecar lifecycle behavior.

## 7. Postmortem for a fleet-scale outage

A strong postmortem is blameless but not accountability-free.

Include:

- Customer impact, scope, duration, safety implications, and SLO/error-budget effect.
- Detection source and detection delay.
- Factual timeline of changes, symptoms, decisions, mitigation, and recovery.
- Technical causal chain rather than blaming an individual.
- Contributing factors such as missing load tests, retry amplification, shared dependency, weak isolation, or poor observability.
- What worked well.
- Assigned, prioritized, due-dated, measurable corrective actions.

A useful causal chain looks like:

```text
valid configuration change
 -> untested edge case
 -> shared dependency overload
 -> retries amplified load
 -> cell isolation was incomplete
 -> health checks remained green
 -> global impact expanded
```

Prefer systemic controls such as rollout guards, bounded retries, quotas, synthetic tests, and dependency isolation over “engineers should be more careful.”

## 8. Behavioral incident story

Use STAR, but include technical judgment, influence, and durable system changes.

Example structure:

> A production-scale migration processed tens of terabytes and left databases in different stages after failure. Re-running blindly risked duplicated work, storage exhaustion, and inconsistent state.
>
> I stopped concurrent recovery attempts, created an authoritative inventory of completed and incomplete units, and decomposed the workflow into discovery, transformation, schema creation, import, validation, and checkpointing. I replaced whole-file processing with streaming transformations and introduced per-database idempotent resume points, logging, and capacity thresholds.
>
> We recovered without restarting completed imports, reduced storage risk, and turned a one-off script into a resumable operational pipeline. The lesson was that restartability is part of correctness: a process that works only when nothing fails is not production automation.

Use only metrics and organizational details you can defend in follow-up questions.
