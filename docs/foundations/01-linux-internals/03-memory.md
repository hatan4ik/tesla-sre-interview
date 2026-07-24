# 3. Virtual Memory, Page Cache, NUMA, Reclaim, and OOM

## 3.1 Why memory incidents are difficult

Linux intentionally uses otherwise idle RAM for caches and delays expensive work until necessary. Therefore:

- Low “free” memory is often healthy.
- Available memory can be high while a constrained cgroup is near OOM.
- A host can have free pages but fail a high-order contiguous allocation.
- Swap activity can hurt latency before the machine appears globally exhausted.
- A process RSS does not equal unique physical memory.
- Page cache can improve performance and still contribute to reclaim stalls.

The correct question is not “How much memory is free?” It is:

> Which memory domain is constrained, what type of pages dominate it, how expensive is reclaim, and which workload is experiencing allocation or fault latency?

## 3.2 Virtual address spaces

Each process sees a virtual address space. Page tables map virtual pages to physical frames or encode nonpresent state.

Typical regions include:

- Executable text and read-only data.
- Writable data and BSS.
- Heap.
- Memory-mapped files and anonymous mappings.
- Shared libraries.
- Thread stacks.
- Kernel-provided helper mappings.

```bash
cat /proc/<pid>/maps
cat /proc/<pid>/smaps_rollup
pmap -x <pid>
```

Virtual size can greatly exceed resident memory. Reserved but untouched address space generally does not consume equivalent physical RAM.

## 3.3 Page tables and the TLB

A virtual address is translated through page-table structures. Walking page tables is expensive, so CPUs cache translations in the Translation Lookaside Buffer.

Performance factors:

- Working-set size relative to TLB reach.
- Page size.
- Process migrations and address-space changes.
- NUMA placement.
- Page-table memory footprint.
- Shootdowns when mappings change across CPUs.

Huge pages increase TLB reach but introduce allocation, fragmentation, compaction, and latency trade-offs.

## 3.4 Demand paging and faults

A page fault is a controlled exception. The kernel determines whether the access is valid and how to satisfy it.

### Minor fault

The required data is already in memory or can be mapped without storage I/O. Examples include copy-on-write and mapping an existing page-cache page.

### Major fault

Storage I/O is required before execution can continue. Major-fault latency can create severe tail latency.

```bash
pidstat -r 1
perf stat -e page-faults,minor-faults,major-faults -p <pid>
```

A high minor-fault rate may be normal for an allocation-heavy workload. A small number of major faults can be devastating for a latency-sensitive service.

## 3.5 Anonymous memory and file-backed memory

- **Anonymous memory:** heap, stacks, and anonymous mappings. It has no file from which clean content can be reread; reclaim may require swap or process termination.
- **File-backed memory:** executable pages, mapped files, and page cache. Clean pages can often be dropped and reread; dirty pages must be written first.

This distinction drives reclaim behavior.

## 3.6 RSS, PSS, USS, and working set

- **RSS:** resident pages mapped into the process, including shared pages counted in each process.
- **PSS:** shared pages proportionally divided among mappings.
- **USS:** memory unique to that process.
- **Working set:** pages actively needed in the relevant time window; it is a behavioral concept, not one universal kernel counter.

For containers, monitoring systems may approximate working set by subtracting some inactive file cache from total usage. Understand the metric definition before using it for sizing or eviction decisions.

## 3.7 Page cache

Linux caches file data in memory. Reads can be served without device I/O, and buffered writes can complete before data reaches stable storage.

Benefits:

- Read locality.
- Write coalescing.
- Reduced device operations.
- Shared cached pages across processes.

Risks:

- Dirty-page buildup.
- Reclaim competition.
- Writeback bursts.
- Double caching when applications also maintain large caches.
- Misleading process-versus-cgroup accounting.

Do not “fix” low free memory by routinely dropping caches. That discards useful state, can create an I/O storm, and hides the root cause.

## 3.8 Buffered, direct, and memory-mapped I/O

### Buffered I/O

Uses page cache and normal read/write paths. It is the default and often best choice.

### Direct I/O

Attempts to transfer between application buffers and storage without normal page-cache data caching. It can reduce double caching and provide application-controlled buffering, but requires alignment and does not eliminate all kernel work.

### mmap

Maps file pages into the process address space. Accesses occur through page faults and memory loads/stores. It simplifies some access patterns but can turn storage latency into fault latency inside arbitrary instruction paths.

The choice should be based on workload locality, durability requirements, memory budget, and observability—not ideology.

## 3.9 Dirty pages and writeback

Buffered writes dirty page-cache pages. The kernel writes them back according to thresholds, age, pressure, and filesystem behavior.

Potential failure mode:

1. Workload writes faster than storage can persist.
2. Dirty pages accumulate.
3. Background writeback cannot keep up.
4. Writers are throttled or forced into direct reclaim/writeback paths.
5. Tail latency spikes.

Evidence:

```bash
watch -n1 'grep -E "Dirty|Writeback|MemAvailable" /proc/meminfo'
vmstat 1
iostat -xz 1
cat /proc/pressure/io
```

Changing dirty ratios can move pain rather than solve it. The durable fix may be storage throughput, write shaping, batching, partitioning, or workload backpressure.

## 3.10 Physical allocation: buddy and slab allocators

The page allocator manages physical memory in blocks of different orders. High-order allocations require contiguous page ranges and may trigger reclaim or compaction.

Kernel objects are commonly allocated through slab-family caches. Slab growth can indicate legitimate object use or leaks involving dentries, inodes, networking objects, or driver allocations.

```bash
slabtop
cat /proc/buddyinfo
cat /proc/pagetypeinfo
```

A system can have many free base pages yet fail a contiguous high-order allocation because of fragmentation.

## 3.11 Reclaim

When memory is scarce, Linux tries to free pages.

### Background reclaim

Kernel threads reclaim proactively as watermarks are crossed.

### Direct reclaim

The allocating task itself performs reclaim. This directly adds latency to the request path.

### Compaction

The kernel moves pages to create contiguous ranges for large allocations. Compaction can consume CPU and stall allocations.

Evidence:

```bash
vmstat 1
cat /proc/vmstat | grep -E 'pgscan|pgsteal|allocstall|compact|workingset'
cat /proc/pressure/memory
```

Key interpretation:

- High scan with low steal efficiency suggests difficult reclaim.
- Allocation stalls indicate request-path impact.
- PSI shows how much time tasks lose because memory is unavailable.

## 3.12 Pressure Stall Information

PSI measures time during which tasks are stalled due to CPU, memory, or I/O pressure.

```bash
cat /proc/pressure/cpu
cat /proc/pressure/memory
cat /proc/pressure/io
```

`some` means at least one task is stalled. `full` means all non-idle tasks in the scope are stalled simultaneously. Cgroup PSI allows workload-specific pressure analysis.

PSI is valuable because it measures experienced pressure rather than only resource occupancy.

## 3.13 Swap

Swap extends reclaim options for anonymous memory. It can protect file cache and avoid abrupt OOM, but faulting swapped pages back in adds latency.

Important distinctions:

- Swap configured versus actively thrashing.
- Occasional cold-page eviction versus sustained swap-in/out.
- Host swap policy versus container memory semantics.
- Disk-backed swap versus compressed in-memory mechanisms.

```bash
swapon --show
vmstat 1
sar -W 1
cat /proc/meminfo | grep -E 'Swap|Anon|Active|Inactive'
```

“Never use swap” and “swap solves memory shortage” are both weak blanket statements. Decide based on workload latency, overcommit, failure policy, and reclaim behavior.

## 3.14 Overcommit

Linux may allow virtual allocations whose total exceeds physical memory plus swap. Allocation success does not guarantee future physical backing.

Overcommit can be useful because many mappings are sparse or never fully touched. It becomes dangerous when workloads collectively realize their reservations.

Inspect:

```bash
sysctl vm.overcommit_memory
sysctl vm.overcommit_ratio
cat /proc/meminfo | grep -E 'CommitLimit|Committed_AS'
```

Databases and memory-intensive platforms may require explicit policy and admission control rather than relying on optimistic allocation.

## 3.15 NUMA

On NUMA systems, memory access latency and bandwidth depend on which CPU socket owns the memory.

First-touch placement often means pages are allocated on the node local to the CPU that first writes them. Problems arise when:

- Initialization threads allocate memory on one node and worker threads run elsewhere.
- CPU affinity changes without memory rebalance.
- A local node is exhausted while remote memory remains available.
- Devices and application threads are on different NUMA nodes.

```bash
numactl --hardware
numastat
numastat -p <pid>
lscpu -e=CPU,NODE,SOCKET,CORE
```

NUMA tuning is an end-to-end placement problem involving CPU, memory, NIC, storage, and container topology.

## 3.16 Huge pages

### Transparent Huge Pages

The kernel may promote mappings automatically. Benefits include fewer TLB misses. Costs include compaction, latency variation, and memory waste for sparse use.

### Explicit huge pages

Applications reserve and manage huge pages deliberately. This provides predictability but reduces flexibility.

Do not enable or disable THP based solely on folklore. Benchmark representative allocation, fault, and latency behavior.

```bash
grep -R . /sys/kernel/mm/transparent_hugepage 2>/dev/null
cat /proc/meminfo | grep -i huge
```

## 3.17 OOM behavior

OOM occurs when the relevant memory domain cannot satisfy allocations and reclaim cannot make progress.

Possible domains:

- Global host OOM.
- Cgroup-local OOM.
- NUMA-node or cpuset constraint.
- High-order allocation failure without global exhaustion.

The kernel selects a victim using badness heuristics influenced by memory usage, adjustment scores, and constraints. The largest process is not guaranteed to be killed.

Evidence:

```bash
journalctl -k | grep -i -A30 -B10 'out of memory\|oom-kill\|killed process'
cat /proc/<pid>/oom_score
cat /proc/<pid>/oom_score_adj
```

For cgroups:

```bash
cat /sys/fs/cgroup/<group>/memory.events
cat /sys/fs/cgroup/<group>/memory.current
cat /sys/fs/cgroup/<group>/memory.max
cat /sys/fs/cgroup/<group>/memory.high
cat /sys/fs/cgroup/<group>/memory.stat
```

## 3.18 cgroup v2 memory controls

Important controls:

- `memory.current`: current accounted usage.
- `memory.max`: hard limit.
- `memory.high`: throttling/reclaim boundary intended to apply pressure before hard OOM.
- `memory.low`: best-effort protection.
- `memory.min`: stronger protection.
- `memory.swap.max`: swap limit.
- `memory.events`: high, max, OOM, and kill events.

Hard limits create a cliff. `memory.high` can provide an earlier pressure signal and force reclaim, but it must be tuned and monitored.

In Kubernetes, request/limit/QoS behavior interacts with node eviction and runtime cgroups. A pod may be killed by its cgroup, evicted by kubelet, or affected by node-level OOM; distinguish them from evidence.

## 3.19 Memory leaks versus cache growth

A rising memory graph may be:

- True unreachable-object leak.
- Intentionally retained application cache.
- Page cache.
- Allocator fragmentation.
- Memory-mapped files.
- Kernel slab growth.
- Queue backlog.
- Connection or session growth.

Investigation sequence:

1. Determine which accounting domain grows.
2. Split anonymous, file, shared, and kernel memory.
3. Compare logical workload cardinality with memory.
4. Inspect allocator and runtime metrics.
5. Use heap profiles where appropriate.
6. Check whether memory returns after workload drops.

Restarting proves only that restart releases memory; it does not identify the source.

## 3.20 Incident scenario: container OOM on a healthy node

### Symptoms

- One pod restarts with `OOMKilled`.
- Node has tens of gigabytes available.
- Application RSS appears below the configured limit.

### Investigation

1. Read cgroup `memory.events`.
2. Inspect `memory.stat` for anonymous, file, slab, and socket memory.
3. Verify whether metrics include page cache and kernel memory.
4. Examine workload concurrency and queue depth.
5. Check limit changes and runtime allocator behavior.

### Likely cause

The cgroup includes page cache and socket/kernel accounting not visible in a process-only RSS dashboard. Burst traffic increases buffers and cache, crossing `memory.max`.

### Mitigation

Reduce concurrency, raise the validated limit, drain traffic, or reduce buffered data.

### Prevention

Monitor cgroup current/high/max events and memory composition; load-test bursts; size from peak working set plus bounded overhead rather than average RSS.

## 3.21 Incident scenario: latency spikes without OOM

### Symptoms

- p99 rises periodically.
- No OOM kills.
- CPU usage rises moderately.
- Storage throughput spikes.

### Investigation

- Memory PSI rises.
- `allocstall` and reclaim counters increase.
- Dirty pages reach thresholds.
- Major faults or swap-ins occur.

The host is spending request time reclaiming and writing pages. OOM is a late-stage outcome; user impact begins earlier.

## 3.22 Interview questions

### Why is low free memory not necessarily bad?

Linux uses free RAM as page cache and reclaimable kernel caches. `MemAvailable`, pressure, reclaim efficiency, and workload latency are more meaningful than raw free pages.

### Can a machine OOM with free memory?

Yes. The constrained allocation may be inside a cgroup, cpuset/NUMA node, or high-order contiguous request. Global free pages do not guarantee that the eligible domain can satisfy it.

### Difference between RSS and working set?

RSS counts resident mapped pages and may double-count shared memory across processes. Working set approximates pages actively required in a time window. They answer different questions.

### Why can page cache cause latency?

Under pressure, cache eviction, dirty writeback, direct reclaim, and refaults can enter the application path. Cache is beneficial until its management competes with the SLO-critical workload.

### What should you capture after OOM?

Kernel OOM report, cgroup memory events and stats, limits, process memory composition, workload cardinality, recent configuration changes, and node pressure. Preserve the exact OOM domain and victim selection context.

## 3.23 Hands-on labs

1. Allocate sparse virtual memory, then touch pages gradually and compare VSZ, RSS, and faults.
2. Generate file-cache pressure and observe `MemFree`, `MemAvailable`, cache, and reclaim.
3. Apply `memory.high` and `memory.max` to a test cgroup; compare throttling and OOM behavior.
4. Trigger controlled dirty-page buildup against a slow loop device and observe writeback latency.
5. Compare local and remote NUMA allocation performance on a multi-node lab host.
6. Capture and interpret an intentional cgroup OOM report.

## 3.24 Principal-level summary

> I debug memory by identifying the constrained domain and memory type, then measuring experienced pressure. I separate anonymous memory, file cache, kernel objects, socket buffers, and allocator fragmentation. I inspect reclaim, refault, writeback, NUMA locality, and cgroup events before changing limits. OOM is usually the final symptom; direct reclaim and pressure-driven tail latency often damage the SLO much earlier.
