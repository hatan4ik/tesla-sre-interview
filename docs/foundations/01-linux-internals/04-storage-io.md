# 4. VFS, Filesystems, Block I/O, NVMe, and Latency

## 4.1 The end-to-end I/O path

A storage request can traverse multiple queues and transformations:

```text
application
  -> userspace runtime / libc
  -> syscall
  -> VFS
  -> filesystem
  -> page cache or direct-I/O path
  -> block layer
  -> device mapper / encryption / RAID
  -> driver
  -> controller queue
  -> device firmware
  -> media
```

In virtualized or cloud environments, add hypervisor, network storage, remote replication, and provider-side throttling. A single “disk latency” metric rarely identifies the real queue.

## 4.2 VFS objects

The Virtual Filesystem Switch provides common abstractions across filesystems.

- **Superblock:** mounted filesystem instance and its metadata.
- **Inode:** metadata for a filesystem object, excluding its name.
- **Dentry:** a name-to-inode association used during pathname lookup.
- **File object:** an open instance with offset, flags, and operations.
- **File descriptor:** process-local integer referencing an open file object.

Important implication:

> A filename is not the file. Names live in directory entries; open descriptors reference kernel file objects and underlying inodes.

This explains why a process can continue writing to a deleted file: unlink removes a directory name, but storage is released only when no links and no open references remain.

```bash
lsof +L1
find /proc/*/fd -lname '*deleted*' 2>/dev/null
```

## 4.3 Path lookup

Resolving `/a/b/c` involves walking directory components, consulting dentry and inode caches, checking mount points, permissions, symlinks, and security hooks.

Path-heavy workloads can be limited by metadata rather than data throughput. Millions of small files may create:

- Dentry/inode cache pressure.
- Lock contention.
- Random metadata I/O.
- Directory scaling problems.
- Long backup and traversal times.

Measure operation shape, not only bytes per second.

## 4.4 File descriptors and limits

Descriptor exhaustion can affect a process or the host.

```bash
ulimit -n
cat /proc/<pid>/limits
ls /proc/<pid>/fd | wc -l
cat /proc/sys/fs/file-nr
sysctl fs.file-max
```

Failure modes:

- Application leak of sockets/files.
- Accept loop stops with `EMFILE`.
- Supervisor limit differs from interactive shell limit.
- Host-wide file table exhaustion.
- Descriptor inherited unintentionally across `exec` because close-on-exec was not set.

A robust server reserves operational capacity and handles exhaustion without spinning or dropping all diagnostics.

## 4.5 Filesystem durability

A successful `write()` to buffered I/O usually means data reached kernel memory, not stable media. Durability may require `fsync()`, `fdatasync()`, ordered metadata operations, and correct handling of parent directories during rename/create workflows.

Storage stacks can lie or weaken guarantees through volatile caches, controller behavior, network failures, or incorrect flush implementation.

Interview principle:

> Durability is an end-to-end contract from application protocol through filesystem, block layer, controller, replication, and power-failure behavior.

## 4.6 Journaling

Journaling filesystems record enough metadata—and depending on mode, data—to recover consistency after a crash. Journaling does not automatically provide application-level transaction semantics.

Questions to ask:

- What ordering does the filesystem guarantee?
- When is a rename durable?
- Is the journal on the same failure domain?
- Are device write caches protected?
- Does the application issue required sync operations?

## 4.7 ext4 and XFS: operational comparison

Both are mature general-purpose filesystems, but they have different implementation and scaling characteristics.

A practical choice should consider:

- Workload file sizes and metadata rate.
- Parallelism and allocation patterns.
- Online growth/shrink requirements.
- Repair and recovery tooling.
- Snapshot/volume layer.
- Distribution support and operational familiarity.

Avoid superficial claims such as “XFS is always faster” or “ext4 is safer.” Benchmark representative workloads and failure recovery.

## 4.8 Copy-on-write and snapshot layers

Copy-on-write appears in filesystems, device-mapper snapshots, virtual disks, and container layers. Benefits include snapshots and space efficiency. Costs can include:

- Write amplification.
- Fragmentation.
- Metadata pressure.
- Unpredictable first-write latency.
- Deep layer chains.

A database writing through a container overlay onto a snapshotting volume can encounter several COW layers. Understand the complete stack.

## 4.9 OverlayFS and container storage

Container writable layers commonly use OverlayFS semantics:

```text
lower read-only image layers
          +
upper writable layer
          =
merged mount visible to container
```

On first modification, a lower-layer file may be copied up. Container image layout and write patterns can therefore affect startup and I/O.

Best practice:

- Keep mutable application data on explicit volumes.
- Treat writable image layers as ephemeral.
- Avoid large database writes to overlay layers.
- Monitor inode use as well as bytes.

## 4.10 Inodes and capacity

A filesystem can run out of inodes while free bytes remain.

```bash
df -h
df -i
find /path -xdev -type f | wc -l
```

Small-file storms from logs, temporary files, package caches, or application bugs can exhaust metadata capacity. Alert on inode consumption and file-creation rate.

## 4.11 Page cache and read-ahead

Sequential access may trigger read-ahead, while random access benefits less. Excessive read-ahead can waste bandwidth and cache; insufficient read-ahead can underutilize a high-latency device.

The kernel and application may both prefetch. Evaluate:

- Access pattern.
- Device latency and bandwidth.
- Working-set size.
- Cache pollution.
- Tail behavior after cold start or failover.

## 4.12 Block layer and multi-queue

Modern Linux block I/O uses multiple hardware/software queues to scale with CPUs and high-parallelism devices. Requests may be merged, scheduled, dispatched, completed, and accounted.

Key concepts:

- Queue depth.
- In-flight operations.
- Request size.
- Sequential versus random access.
- Read/write mix.
- Flush and barrier operations.
- Device and controller parallelism.

A device can reach latency saturation before advertised throughput or IOPS.

## 4.13 I/O schedulers

Schedulers mediate request ordering and fairness. The best policy depends on device type, virtualization, and workload.

```bash
cat /sys/block/<device>/queue/scheduler
cat /sys/block/<device>/queue/nr_requests
```

Do not select a scheduler from a generic tuning guide. Some cloud volumes are virtual abstractions where provider-side queues dominate.

## 4.14 Understanding iostat

```bash
iostat -xz 1
```

Interpret fields in context:

- Operations per second.
- Throughput.
- Average request size.
- Queue depth.
- Await/latency.
- Utilization.

Cautions:

- Averaging hides tail latency.
- Device-mapper layers can double-count or obscure attribution.
- A parallel NVMe device can service requests while showing 100% “utilization”; that does not mean the same thing as one spinning disk.
- Cloud throttling may occur outside the guest.

Correlate with application latency and per-cgroup/process I/O.

## 4.15 Little’s Law for I/O

For a stable system:

```text
concurrency ≈ throughput × latency
```

If latency rises while throughput is fixed, in-flight work grows. If queue depth is capped, throughput falls. This is a powerful way to reason about storage saturation and application concurrency.

Example:

- 10,000 operations/s.
- 2 ms average latency.
- Roughly 20 operations in flight.

At 20 ms latency, the same throughput requires roughly 200 in flight. If the stack allows only 64, throughput collapses or callers queue elsewhere.

## 4.16 NVMe

NVMe supports many queues and high parallelism. Operational issues include:

- Queue-depth mismatch.
- Thermal throttling.
- Firmware errors or resets.
- NUMA distance between device and CPU.
- Interrupt affinity.
- Device wear and media errors.
- Namespace configuration.

```bash
nvme list 2>/dev/null
nvme smart-log /dev/nvme0 2>/dev/null
journalctl -k | grep -iE 'nvme|reset|timeout|I/O error'
```

A low average latency does not rule out firmware reset pauses that destroy p99.99.

## 4.17 RAID and replication

RAID changes capacity, availability, and write behavior.

Questions:

- What failures are tolerated?
- What is the rebuild impact?
- Does parity create read-modify-write amplification?
- Is the controller cache protected?
- Does replication share power, rack, zone, or control-plane failure domains?

RAID is not backup. Replication can faithfully replicate corruption or deletion.

## 4.18 Device mapper, LVM, and encryption

A typical enterprise path may include:

```text
filesystem -> logical volume -> thin pool -> encryption -> multipath -> device
```

Each layer introduces metadata, queues, failure modes, and observability. Thin provisioning requires monitoring both data and metadata capacity. Running out of thin-pool metadata can be catastrophic even with apparent free logical space.

```bash
lsblk -o NAME,TYPE,FSTYPE,SIZE,MOUNTPOINTS
lvs -a -o +devices,data_percent,metadata_percent
 dmsetup ls --tree
```

## 4.19 Network filesystems

NFS and similar systems combine storage and network failure semantics. D-state accumulation, long hangs, stale handles, retransmissions, server overload, and mount-option behavior are common.

Questions:

- Hard or soft mount semantics?
- Timeouts and retry behavior?
- Locking and cache consistency?
- What happens during server failover?
- Can a failed mount block unrelated host operations?

Never choose soft failure semantics merely to avoid hangs without understanding data-integrity consequences.

## 4.20 Disk-full failure modes

“Disk full” can mean:

- No data blocks.
- No inodes.
- Reserved blocks unavailable to non-root.
- Thin-pool data or metadata exhausted.
- Snapshot capacity exhausted.
- Project/user quota reached.
- Log filesystem full while data filesystem is healthy.
- Deleted-open files consuming space.

Runbook:

```bash
df -hT
df -i
lsof +L1
findmnt
lvs -a -o +data_percent,metadata_percent 2>/dev/null
journalctl --disk-usage
```

Do not immediately delete arbitrary files. Identify growth source, preserve evidence, and understand application behavior when files disappear.

## 4.21 I/O latency decomposition

Application-observed latency may include:

- Time waiting in application queue.
- Scheduler delay before issuing I/O.
- Filesystem locks and metadata work.
- Page-cache miss and reclaim.
- Block queue time.
- Device service time.
- Flush/replication time.
- Scheduler delay before completion handler or application thread runs.

Use tracing to separate issue-to-completion from application wall time.

## 4.22 Useful tools

```bash
# Capacity and topology
lsblk -f
findmnt
df -hT
df -i

# Device performance
iostat -xz 1
sar -d 1
cat /proc/diskstats

# Per-process I/O
pidstat -d 1
cat /proc/<pid>/io

# Files and descriptors
lsof -p <pid>
lsof +L1

# Syscall and latency evidence
strace -f -ttT -e trace=file,read,write,fsync -p <pid>

# Pressure
cat /proc/pressure/io
```

For deeper analysis, use eBPF block-I/O latency histograms, filesystem operation tracing, and off-CPU profiles.

## 4.23 Incident scenario: 100% disk utilization, low throughput

### Symptoms

- Device utilization near 100%.
- Throughput much lower than historical peak.
- Write latency extremely high.
- Queue depth growing.

### Hypotheses

- Small random synchronous writes.
- Flush-heavy workload.
- Device error recovery or reset.
- Cloud IOPS/throughput credit exhaustion.
- RAID rebuild.
- Thin-provisioning metadata pressure.
- Severe write amplification.

### Investigation

1. Check request size and read/write mix.
2. Inspect latency distribution, not only average.
3. Review kernel errors and resets.
4. Inspect volume throttling and backend limits.
5. Correlate with recent workload or configuration changes.
6. Check filesystems, snapshots, RAID, encryption, and device-mapper layers.

### Mitigation

Reduce write concurrency, batch or buffer safely, fail over, expand provisioned performance, pause rebuild/noncritical jobs, or isolate the offending tenant.

## 4.24 Incident scenario: filesystem full after logs deleted

### Symptoms

- Large logs were removed.
- `df` still shows full.
- `du` shows far less usage.

### Root cause

A process still has deleted files open. Directory entries are gone, but inode and blocks remain referenced.

### Resolution

Identify with `lsof +L1`, then rotate/reopen gracefully or restart the specific process. Avoid rebooting the entire node unless required.

## 4.25 Interview questions

### Why can `du` and `df` disagree?

`du` walks named files. `df` reports filesystem block allocation. Deleted-open files, reserved blocks, snapshots, and metadata can create differences.

### What does `fsync()` guarantee?

It requests that file data and required metadata reach the storage stack according to filesystem semantics. End-to-end durability still depends on correct device flush behavior and the application’s handling of directory entries and rename sequences.

### Why can low IOPS still saturate storage?

Operations may be large, synchronous, serialized, flush-heavy, or extremely slow. Saturation depends on service time and concurrency, not IOPS alone.

### Why is average latency insufficient?

A small fraction of multi-second operations can destroy user-facing tail latency while the average remains acceptable. Use histograms and correlate with request traces.

### Why can deleting a file fail to free space?

Open file descriptors continue referencing the inode and blocks until all references close.

## 4.26 Hands-on labs

1. Open a large file, unlink it, and compare `du`, `df`, and `lsof +L1`.
2. Benchmark buffered versus direct I/O with representative block sizes and concurrency.
3. Create many small files and measure metadata behavior and inode consumption.
4. Throttle a loop-backed device and observe dirty writeback, queue depth, and application latency.
5. Compare synchronous writes with batched fsync and document durability trade-offs.
6. Build an I/O latency histogram with an eBPF tool and compare it with `iostat` averages.

## 4.27 Principal-level summary

> I debug storage as an end-to-end queueing system. I distinguish application queueing, filesystem work, cache and reclaim, block-layer delay, device service time, and external replication or throttling. I characterize operation size, sync semantics, concurrency, locality, and tail latency before tuning. Capacity, durability, and failure recovery are separate dimensions; a fast benchmark does not prove correct crash behavior.
