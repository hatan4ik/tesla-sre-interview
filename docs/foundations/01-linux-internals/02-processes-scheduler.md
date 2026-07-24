# 2. Processes, Threads, Scheduling, Interrupts, and Load

## 2.1 Task model

The Linux scheduler operates on tasks. A traditional process and a thread are both represented by task structures; what differs is which resources they share.

A task can share or own:

- Virtual address space.
- File-descriptor table.
- Filesystem context.
- Signal handlers.
- Credentials.
- Namespace memberships.
- Thread-group identity.

`clone()` exposes these sharing choices. Higher-level APIs such as `fork()` and thread creation are built on related kernel mechanisms.

```text
Process
  thread-group ID
  shared address space
  shared open files
  shared signal dispositions
       |
       +-- thread/task A
       +-- thread/task B
       +-- thread/task C
```

## 2.2 Process creation

### fork

`fork()` creates a child with a logically separate address space, but pages are initially shared through copy-on-write. Page-table work and later write faults still have costs; “fork is cheap” is only conditionally true.

Large-memory processes can experience:

- Page-table duplication overhead.
- Copy-on-write amplification after the child or parent writes.
- Latency spikes from memory allocation and TLB activity.
- Unexpected memory pressure after a supposedly harmless fork.

This matters for databases, language runtimes, and backup patterns.

### exec

`execve()` replaces the calling task’s program image while retaining the process identity and selected inherited resources. The kernel loads the executable, maps segments and libraries, constructs the initial stack, and begins at the new entry point.

Security depends on controlling environment, file descriptors, interpreter paths, credentials, and executable provenance.

### exit and zombies

A terminating child retains a minimal record until its parent collects the exit status. That record is a zombie. Zombies consume PID table entries, not full process memory. A growing zombie count indicates broken child reaping, often by PID 1 or a supervisor.

```bash
ps -eo pid,ppid,state,comm | awk '$3 ~ /Z/'
```

## 2.3 Task states

Common `ps` states:

- `R`: runnable or running.
- `S`: interruptible sleep.
- `D`: uninterruptible sleep, commonly waiting inside the kernel.
- `T`: stopped or traced.
- `Z`: zombie.

`D` state is important because tasks generally do not respond to ordinary signals until the kernel operation completes. Common causes include storage, remote filesystems, device drivers, or blocked kernel paths.

Do not conclude “disk problem” from `D` state alone. Capture kernel stacks:

```bash
ps -eo state,pid,ppid,wchan:32,comm | awk '$1 == "D"'
cat /proc/<pid>/stack
sysrq-trigger documentation should be reviewed before using blocked-task dumps in production
```

## 2.4 Context switches

A context switch saves execution state for one task and restores another. Cost includes more than register operations:

- Scheduler bookkeeping.
- Cache and branch-predictor disruption.
- Possible address-space and TLB effects.
- Lock contention and cross-CPU wakeups.
- Lost locality when a task migrates.

Two broad classes:

- **Voluntary:** task blocks or yields.
- **Involuntary:** preempted by scheduler.

```bash
pidstat -w 1
vmstat 1
perf stat -e context-switches,cpu-migrations,page-faults -p <pid>
```

High context-switch rate is not automatically bad. Event-driven servers may switch frequently and perform well. Correlate switches with scheduler latency, CPU use, throughput, and tail latency.

## 2.5 Scheduler classes

Linux supports multiple scheduling classes, conceptually ordered by policy. Production systems commonly encounter:

- Normal fair scheduling.
- Batch and idle policies.
- Real-time FIFO and round-robin.
- Deadline scheduling for explicitly modeled real-time workloads.

Real-time scheduling is dangerous when misused. A runaway high-priority task can starve ordinary work, including operational access. Use resource reservations, watchdogs, affinity, and tested rollback.

## 2.6 Fair scheduling and run queues

Normal tasks are distributed across per-CPU run queues. The scheduler tries to allocate CPU fairly according to task weights while balancing locality and load.

Key points:

- Nice values influence weight; they are not percentages.
- Per-CPU queues reduce global lock contention.
- Load balancing can migrate tasks, trading fairness for cache locality.
- Wakeup placement matters for latency-sensitive threads.
- Cgroup CPU controls introduce hierarchical competition.

The exact internal implementation evolves; interview answers should focus on invariants and observable behavior rather than memorizing one kernel version’s tree structure.

## 2.7 CPU utilization versus saturation

Utilization measures time spent busy. Saturation measures queued demand.

A CPU can be:

- Highly utilized but not saturated: work completes without meaningful queueing.
- Moderately utilized but latency-sensitive tasks are saturated on one hot core.
- Apparently idle while work waits on I/O, locks, throttling, or unavailable CPUs.

Signals:

```bash
mpstat -P ALL 1
pidstat -u -t 1
cat /proc/loadavg
cat /proc/pressure/cpu
top -H
```

Look for:

- Uneven per-CPU use.
- Run-queue depth.
- CPU pressure.
- Steal time in virtualized environments.
- Guest time.
- Softirq concentration.
- Cgroup throttling.

## 2.8 Load average

Linux load average tracks tasks that are runnable plus tasks in uninterruptible sleep. It is not a direct CPU utilization metric.

Consequences:

- High load with idle CPU can mean many `D`-state tasks.
- A short burst can remain visible because load is exponentially smoothed.
- A load of 8 has different meaning on a 2-CPU system and a 128-CPU system.
- Container-visible load may reflect host-wide behavior depending on environment and tooling.

Interview answer:

> I interpret load average alongside CPU count, runnable tasks, D-state tasks, PSI, and per-resource latency. I do not page on load average alone.

## 2.9 CPU affinity and NUMA locality

Affinity can improve cache and NUMA locality, but static pinning can also create stranded capacity or hot cores.

```bash
taskset -cp <pid>
numactl --hardware
numastat -p <pid>
cat /proc/<pid>/status | grep -E 'Cpus_allowed|Mems_allowed'
```

Use cases:

- Pin latency-sensitive data-plane threads near the NIC queue they serve.
- Keep memory allocation local to the CPUs that consume it.
- Isolate noisy system work from dedicated workloads.

Risks:

- IRQ and application threads pinned to the same CPU.
- Containers receiving CPU quotas without topology awareness.
- Thread pools larger than effective CPU allocation.
- Cross-socket memory access raising latency and consuming interconnect bandwidth.

## 2.10 CPU cgroup controls

Important controls include:

- CPU weight: relative share during contention.
- CPU quota and period: hard bandwidth limit.
- Cpuset: allowed CPUs and memory nodes.

Quota can produce counterintuitive latency. A multithreaded process may consume its entire period quota quickly, then be throttled until the next period despite idle CPUs elsewhere.

Evidence:

```bash
cat /sys/fs/cgroup/<group>/cpu.stat
cat /sys/fs/cgroup/<group>/cpu.max
cat /sys/fs/cgroup/<group>/cpu.pressure
```

Look for throttled periods and throttled time. In Kubernetes, CPU limits map to quota semantics on many setups. Latency-sensitive services may be harmed by overly tight CPU limits even when average usage appears low.

## 2.11 Interrupt handling

### Hardware interrupts

Devices signal CPUs through interrupts. The kernel acknowledges the event and defers substantial processing.

### Softirqs

Network receive/transmit, timers, and other deferred work can run in softirq context. When the kernel cannot complete it promptly, per-CPU `ksoftirqd` threads process the backlog.

```bash
watch -n1 'grep -E "NET_RX|NET_TX|TIMER" /proc/softirqs'
watch -n1 'grep -E "eth|ens|enp|nvme" /proc/interrupts'
```

### IRQ affinity

Modern NICs expose multiple queues. RSS distributes flows across receive queues, and each queue can map to an interrupt vector. Poor affinity can overload one CPU while others remain idle.

End-to-end locality can include:

```text
NIC RX queue -> IRQ CPU -> softirq CPU -> application CPU -> NUMA-local memory
```

Blindly spreading everything across all CPUs is not always optimal. Validate with per-queue drops, per-CPU softirq, cache behavior, and latency.

## 2.12 Frequency scaling, thermal throttling, and steal

A host can show 100% busy CPU yet deliver less compute because of:

- Reduced clock frequency.
- Thermal or power limits.
- Hypervisor steal time.
- SMT sibling interference.
- CPU allocation changes.

Useful evidence varies by platform:

```bash
lscpu
mpstat 1
cat /proc/cpuinfo | grep -m1 'cpu MHz'
find /sys/devices/system/cpu/cpufreq -type f 2>/dev/null | head
journalctl -k | grep -iE 'thermal|thrott|mce|hardware error'
```

Cloud environments may hide physical details. Compare achieved work per CPU-second, not only utilization.

## 2.13 Lock contention and futexes

Most userspace mutexes avoid syscalls when uncontended. Under contention, futex operations allow waiters to sleep and be awakened by the kernel.

Symptoms:

- CPU not fully utilized.
- High latency and low throughput.
- Threads blocked in futex waits.
- One owner thread hot or stalled.

Evidence:

```bash
strace -f -e futex -p <pid>
perf lock record -- <command>
perf lock report
perf sched record -p <pid> -- sleep 10
perf sched latency
```

The fix may be reducing critical-section duration, sharding locks, avoiding oversubscribed thread pools, or eliminating a serialized dependency. Adding CPUs can worsen contention.

## 2.14 Scheduler latency

CPU utilization does not reveal how long a runnable task waited before executing. Scheduler latency is critical for tail-sensitive services.

Potential causes:

- Oversubscription.
- Long-running nonpreemptible kernel sections.
- Real-time task interference.
- IRQ/softirq storms.
- Cgroup throttling.
- CPU affinity mistakes.
- VM steal.

Use `perf sched`, eBPF run-queue tools, PSI, and application latency correlation.

## 2.15 Incident scenario: high p99, normal average CPU

### Symptoms

- API p50 unchanged.
- p99 increases from 100 ms to 2 s.
- Host average CPU is 45%.
- No increase in dependency latency.

### Investigation

1. Break down per-CPU utilization.
2. Inspect thread-level CPU and run-queue delay.
3. Check cgroup `cpu.stat` for throttling.
4. Check IRQ and softirq distribution.
5. Compare slow requests with CPU period boundaries.

### Likely finding

The container has a low CPU quota. A burst of parallel work consumes quota early, and all threads are throttled until the next period. Node-level average CPU remains low.

### Mitigation

Raise or remove the inappropriate limit, reduce concurrency, or allocate a dedicated class of service.

### Prevention

Alert on throttled time and CPU PSI; load-test with bursty concurrency; define resource policies by workload latency class.

## 2.16 Incident scenario: high load, idle CPU

### Symptoms

- Load average 80 on a 32-vCPU node.
- CPU idle 70%.
- Application requests hang.

### Investigation

```bash
ps -eo state,pid,wchan:32,comm | sort
cat /proc/pressure/io
cat /proc/pressure/memory
iostat -xz 1
```

Capture blocked task stacks. A cluster of tasks in an NFS or block-driver path points away from CPU.

### Mitigation

Fence or isolate the failing dependency, stop new work, remount or fail over according to a tested runbook, and preserve stack evidence.

### Prevention

Bound remote-filesystem dependencies, use timeouts and circuit breakers, alert on D-state counts and I/O PSI, and test failure behavior.

## 2.17 Interview questions

### What is the difference between a process and a thread in Linux?

Both are schedulable tasks. Threads in a process share selected resources such as address space and file descriptors, while separate processes generally have distinct resource contexts. The distinction is implemented through clone-time sharing flags rather than two unrelated kernel object types.

### Why can more threads reduce throughput?

Oversubscription increases scheduling, cache misses, lock contention, memory footprint, and queueing. Optimal concurrency is bounded by available parallelism and downstream capacity.

### What does `D` state mean?

The task is in uninterruptible sleep in a kernel path. It is commonly waiting for I/O but the kernel stack is the authoritative evidence. Signals are generally acted on only after the wait completes.

### How can CPU be idle while an application is CPU-throttled?

Cgroup quota can throttle a group after it consumes its allowed bandwidth for the period even when other host CPUs are idle. Check cgroup throttling counters and effective CPU allocation.

### Why is per-CPU analysis important?

A global average hides hot cores, IRQ concentration, affinity errors, single-thread bottlenecks, and NUMA-local saturation.

## 2.18 Hands-on labs

1. Run a CPU-bound workload with different nice values and compare share under contention.
2. Apply a cgroup CPU quota to a multithreaded workload and observe throttled time and tail latency.
3. Pin a workload and a synthetic interrupt-heavy process to the same CPU, then move one and compare latency.
4. Create lock contention in a test program and analyze on-CPU and off-CPU behavior.
5. Simulate many blocked tasks on a slow filesystem and compare load average, CPU utilization, and I/O PSI.

## 2.19 Principal-level summary

> I separate CPU utilization from runnable demand, scheduler delay, and cgroup throttling. I inspect per-CPU behavior because global averages hide hot queues and affinity problems. For latency incidents, I correlate run-queue delay, throttled time, interrupts, softirqs, lock waits, and NUMA placement with request traces. I tune only after locating the actual queue.
