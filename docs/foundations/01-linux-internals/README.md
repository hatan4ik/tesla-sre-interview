# Chapter 1 — Linux Internals for Staff and Principal SREs

This chapter treats Linux as a production system rather than a collection of shell commands. The objective is to reason from symptoms to kernel mechanisms, quantify bottlenecks, and explain trade-offs at Staff/Principal level.

## Learning outcomes

By the end of the chapter, you should be able to:

- Trace a request from userspace through syscalls, scheduling, memory, networking, storage, and hardware.
- Explain why a machine can show high load with idle CPUs, free memory with OOM kills, or low disk utilization with terrible latency.
- Distinguish process, thread, namespace, cgroup, container, and virtual-machine isolation.
- Diagnose CPU saturation, memory pressure, I/O stalls, packet loss, lock contention, and kernel-level latency.
- Connect Linux behavior to Kubernetes QoS, eviction, networking, storage, and node reliability.
- Build a hypothesis-driven incident plan using `/proc`, PSI, `perf`, `strace`, `ss`, `ethtool`, `iostat`, and eBPF.
- Give concise interview answers that move from mechanism to evidence, mitigation, and prevention.

## Chapter map

1. [Architecture, boot, privilege boundaries, and syscalls](01-architecture-boot-syscalls.md)
2. [Processes, threads, scheduling, interrupts, and load](02-processes-scheduler.md)
3. [Virtual memory, page cache, NUMA, reclaim, and OOM](03-memory.md)
4. [VFS, filesystems, block I/O, NVMe, and latency](04-storage-io.md)
5. [Networking, namespaces, cgroups, containers, and security](05-networking-containers-security.md)
6. [Observability and production debugging](06-observability-debugging.md)
7. [Interview drills, incident scenarios, and hands-on labs](07-interview-labs.md)

## Staff-level mental model

```text
Application intent
      |
      v
Userspace runtime and libraries
      |
      v
System-call boundary
      |
      +--> scheduler and CPU
      +--> virtual memory and page cache
      +--> VFS and block layer
      +--> socket and network stack
      +--> security hooks and resource controls
      |
      v
Driver, device, firmware, hypervisor, hardware
```

When debugging, avoid jumping directly from a high-level symptom to a tuning parameter. Walk the path and locate the queue where work is accumulating.

## Universal production-debugging sequence

1. **Define impact.** Which users, workloads, nodes, zones, and SLOs are affected?
2. **Establish time.** When did it start, and what changed immediately before it?
3. **Identify the constrained resource.** CPU time, runnable slots, memory, reclaim bandwidth, I/O queue depth, sockets, conntrack entries, locks, or dependency capacity.
4. **Find the queue.** Run queue, accept queue, socket send/receive queue, dirty-page queue, block queue, allocator wait, futex wait, or application queue.
5. **Separate utilization from saturation.** A device may be lightly utilized but saturated by a serialized path or latency outlier.
6. **Correlate layers.** Application latency, kernel counters, host metrics, container limits, and downstream behavior must tell one coherent story.
7. **Mitigate safely.** Reduce load, shed optional work, isolate a bad tenant, rollback, expand capacity, or bypass a failed dependency.
8. **Preserve evidence.** Capture profiles, counters, logs, and timelines before restarting everything.
9. **Prevent recurrence.** Add guardrails, capacity models, alerts on leading indicators, and a tested runbook.

## Golden signals at the host level

| Resource | Utilization | Saturation / queue | Errors | Latency |
|---|---|---|---|---|
| CPU | busy time by mode | runnable tasks, PSI CPU | machine checks, throttling | scheduler delay, off-CPU time |
| Memory | working set, cache | reclaim, compaction, PSI memory | OOM, allocation failures | major-fault and reclaim latency |
| Disk | throughput, busy time | queue depth, await, PSI I/O | resets, media errors | read/write tail latency |
| Network | packets/bytes | socket queues, backlog, softirq | drops, retransmits | RTT and handshake latency |

## Interview answer pattern

For any Linux question, use this structure:

1. **Mechanism:** explain what the kernel is doing.
2. **Failure mode:** explain how it breaks under load or partial failure.
3. **Evidence:** name the counters and tools that prove or disprove the hypothesis.
4. **Mitigation:** stabilize production without destroying evidence.
5. **Prevention:** propose limits, tests, observability, and architectural changes.

Example:

> A high load average with idle CPU often means tasks are blocked in uninterruptible sleep, commonly waiting on storage or network-backed filesystems. I would split load into runnable and D-state tasks, inspect PSI and per-device latency, then sample kernel stacks before restarting anything. The immediate mitigation may be isolating the failed mount or reducing write pressure; the prevention is bounded I/O, mount timeouts, dependency isolation, and alerts on I/O pressure rather than load alone.

## Scope

The chapter focuses on modern production Linux and containerized workloads. Distribution-specific packaging and desktop administration are intentionally out of scope. Commands may require root privileges and should be tested in a disposable lab before production use.
