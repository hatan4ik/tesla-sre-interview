# 1. Architecture, Boot, Privilege Boundaries, and Syscalls

## 1.1 The production mental model

Linux is a monolithic kernel with loadable modules. “Monolithic” does not mean one undifferentiated program; it means core subsystems execute in the same privileged address space and call one another directly. Scheduling, virtual memory, filesystems, networking, security hooks, and most drivers run in kernel mode.

This design favors performance and a mature internal subsystem model, but a serious kernel or driver defect can compromise or crash the whole host. Isolation inside one kernel is therefore not equivalent to hypervisor isolation.

```text
+----------------------------------------------------------+
| Userspace                                                |
| application | runtime | libc | agents | system services |
+---------------- system-call / exception boundary --------+
| Kernel                                                   |
| scheduler | VM | VFS | block | TCP/IP | LSM | drivers   |
+---------------------- hardware boundary -----------------+
| CPU | RAM | NIC | NVMe | accelerator | firmware         |
+----------------------------------------------------------+
```

A Staff-level answer should connect an application symptom to the kernel queues and state transitions beneath it. “The host is slow” is not a diagnosis. Determine whether work is waiting for CPU, memory reclaim, a lock, storage, a socket, a remote filesystem, or an application dependency.

## 1.2 User mode and kernel mode

Modern CPUs expose privilege levels. Linux primarily uses:

- **User mode:** applications cannot directly access arbitrary physical memory, program devices, or execute privileged instructions.
- **Kernel mode:** the kernel can manage page tables, interrupts, devices, and process state.

Transitions occur through:

- System calls initiated by userspace.
- Exceptions such as page faults and divide-by-zero.
- Hardware interrupts from devices.
- Inter-processor interrupts used for cross-CPU coordination.

A transition is not the same as a process context switch. A process can enter the kernel to execute `read()` and return to the same thread without scheduling another task. Conversely, a scheduler switch may occur while already in kernel mode because the current task blocks or exhausts its time slice.

## 1.3 Linux boot path

```text
Power on
  -> platform firmware (UEFI/BIOS)
  -> boot manager / bootloader
  -> compressed kernel image + initramfs
  -> early kernel initialization
  -> root filesystem discovery and mount
  -> exec PID 1
  -> system services and targets
  -> workload startup
```

### Firmware

Firmware initializes enough CPU, memory, and device state to locate an executable boot target. UEFI usually reads a boot entry from NVRAM and loads an EFI executable from the EFI System Partition.

Failure patterns:

- Missing or corrupted boot entry.
- Firmware cannot see the boot device.
- Secure Boot rejects an unsigned component.
- Hardware or firmware initialization hangs.

### Bootloader

The bootloader selects a kernel and command line, loads the kernel plus initramfs, and transfers control. Kernel arguments can radically change behavior, including root-device selection, cgroup mode, IOMMU, console output, crash dump reservation, and security settings.

Useful evidence:

```bash
cat /proc/cmdline
bootctl status 2>/dev/null || true
grub-editenv list 2>/dev/null || true
```

### Kernel decompression and early initialization

The kernel:

1. Decompresses itself.
2. Establishes early page tables.
3. Discovers CPU and memory topology.
4. Initializes interrupt handling, timers, allocators, and core subsystems.
5. Starts secondary CPUs.
6. Initializes built-in drivers.
7. Mounts the initramfs as a temporary root.

### initramfs

The initramfs contains early userspace needed before the real root filesystem is available. It may load storage, RAID, LVM, encryption, network, and filesystem modules, then locate and mount the actual root.

Classic failure:

> The kernel boots, but the machine drops into an initramfs emergency shell because the root logical volume, encrypted device, or storage controller is unavailable.

Debug sequence:

```bash
cat /proc/cmdline
lsmod
lsblk -f
blkid
dmesg -T | tail -200
```

### PID 1

After mounting the real root, the kernel executes the configured init process, usually `systemd`. PID 1 is special:

- It becomes the ancestor of orphaned processes.
- It must reap children to avoid zombie accumulation.
- Signal behavior differs from ordinary processes.
- Its failure is generally fatal to a normal system.

In containers, the first process in the PID namespace inherits similar responsibilities. Running an application directly as container PID 1 can expose poor signal handling and child reaping. A tiny init process may be justified for workloads that spawn children.

## 1.4 systemd as a dependency and supervision engine

Treat systemd as a graph scheduler, not merely a startup script replacement.

Important unit concepts:

- `Requires=` expresses a hard dependency.
- `Wants=` expresses a weaker dependency.
- `After=` and `Before=` express ordering, not requirement.
- Socket, path, timer, mount, and device units can activate services.
- Cgroups track service processes reliably, unlike PID-file-only supervision.

Commands:

```bash
systemd-analyze time
systemd-analyze critical-chain
systemd-analyze blame
systemctl list-dependencies --reverse some.service
systemctl show some.service -p ActiveState,SubState,MainPID,ControlGroup
journalctl -b -u some.service
```

Interview trap:

> `After=network.target` does not prove usable external connectivity. It only orders the unit after a loosely defined target. A service that requires resolved DNS, routes, credentials, or a reachable dependency needs explicit readiness behavior and retries.

## 1.5 System-call path

A system call is a controlled entry into the kernel. On x86-64, a libc wrapper usually loads a syscall number and arguments into registers and executes the architecture’s syscall instruction. The kernel validates arguments, checks permissions, performs the operation, and returns a result or negative error code.

```text
application
   -> libc wrapper
   -> architecture syscall instruction
   -> syscall entry code
   -> generic kernel subsystem
   -> security hooks / permissions
   -> driver or data structure
   -> return to userspace
```

Examples:

- `openat()` enters the VFS path lookup machinery.
- `read()` may copy from page cache, block on I/O, or read a pipe/socket.
- `connect()` may initiate routing, ARP/neighbor discovery, and TCP handshake state.
- `futex()` coordinates userspace locks by sleeping only when contention occurs.
- `epoll_wait()` blocks until registered file descriptors become ready.

## 1.6 Blocking, nonblocking, and asynchronous execution

These terms are often confused.

- **Blocking:** the calling thread sleeps until progress or timeout.
- **Nonblocking:** the call returns immediately if it cannot progress, often with `EAGAIN`.
- **Synchronous:** completion is observed in the call’s control flow.
- **Asynchronous:** submission and completion are decoupled.

A nonblocking socket plus `epoll` is readiness-based. The application is told that an operation is likely to make progress, then performs it. `io_uring` can support completion-oriented submission for many operations.

## 1.7 Copying and zero-copy paths

A naïve data path may copy bytes repeatedly:

```text
storage -> kernel page cache -> userspace buffer -> kernel socket buffer -> NIC
```

Mechanisms such as `sendfile()`, `splice()`, memory mapping, direct I/O, and NIC offloads can reduce copies or CPU work, but each changes semantics and observability.

Trade-offs:

- Page cache improves locality and absorbs writes but can create reclaim and dirty-page pressure.
- Direct I/O bypasses page cache but imposes alignment and application-cache responsibilities.
- Large offloads reduce packet-processing CPU but can make packet captures look different from wire traffic.
- “Zero copy” often means zero or fewer CPU-mediated copies, not that data never moves.

## 1.8 Page faults as controlled exceptions

Accessing a valid virtual address whose page is not currently mapped triggers a page fault. The kernel may:

- Map an already resident page: minor fault.
- Read data from storage: major fault.
- Allocate a zero-filled anonymous page.
- Perform copy-on-write after `fork()`.
- Deliver `SIGSEGV` for an invalid access.

A fault is not inherently an error. Demand paging depends on faults. The production question is whether fault rate and fault latency are compatible with the workload’s SLO.

## 1.9 Interrupts, softirqs, and deferred work

A device interrupt must be handled quickly. Linux commonly divides work into:

1. A top half that acknowledges the device and schedules deferred work.
2. Deferred processing through softirqs, threaded IRQs, workqueues, or related mechanisms.

Network receive processing can consume substantial softirq CPU. When processing cannot keep up, backlog and drops appear even though application CPU profiles may look normal.

Useful checks:

```bash
cat /proc/interrupts
cat /proc/softirqs
mpstat -P ALL 1
sar -I SUM 1
```

Questions to ask:

- Is one CPU handling most NIC interrupts?
- Is `ksoftirqd` consuming CPU because work exceeded interrupt context budget?
- Are IRQ affinity and receive-side scaling aligned with NUMA locality?
- Are drops occurring in the NIC, driver, kernel backlog, socket, or application?

## 1.10 Kernel modules and drivers

Modules allow subsystems and drivers to be loaded dynamically. In production, module management is a security and reliability boundary.

```bash
lsmod
modinfo <module>
find /sys/module/<module>/parameters -maxdepth 1 -type f -print
```

Risks:

- A driver bug can hang or panic the host.
- Out-of-tree modules complicate upgrades and supportability.
- Dynamic module loading expands attack surface.
- Module parameters can silently change semantics or performance.

For immutable fleets, prefer a tested kernel, controlled module set, reproducible node image, and replacement over long-lived snowflake patching.

## 1.11 Kernel logs and crash evidence

```bash
dmesg -T
journalctl -k -b
journalctl -k -b -1
cat /proc/sys/kernel/tainted
```

Production-grade crash handling may include:

- Persistent journal or remote log shipping.
- pstore for firmware-backed crash records.
- kdump to capture a vmcore using a crash kernel.
- Watchdogs for lockups.
- Machine-check and hardware error telemetry.

Do not treat rebooting as root-cause analysis. A reboot may restore service, but it also destroys volatile evidence.

## 1.12 Staff-level interview drills

### Why can a system call be expensive?

Not because crossing the privilege boundary is always the dominant cost. Expense may come from cache misses, path traversal, permission checks, lock contention, copying, blocking I/O, scheduler delay, TLB disruption, or device latency. Measure the complete path.

### Is a syscall a context switch?

It is a privilege-level transition into the kernel. A scheduler context switch to another task is separate and may or may not occur.

### Why is PID 1 special in a container?

It has namespace-init responsibilities, including orphan adoption and child reaping, and signal semantics can surprise applications. Entrypoints must forward termination signals and reap children or use an init wrapper.

### Why might boot be slow although all individual services start quickly?

The critical path may include dependency serialization, device timeouts, initramfs discovery, mount delays, network-online waits, entropy, or jobs not visible in a simplistic service-duration list. Analyze the dependency chain and timestamps from firmware through userspace.

## 1.13 Hands-on exercises

1. Capture the boot critical path with `systemd-analyze critical-chain` and identify one unnecessary serialized dependency.
2. Use `strace -f -ttT` on a disposable service and classify its time into CPU execution, filesystem waits, network waits, and futex waits.
3. Compare a blocking socket server with an `epoll`-based server under thousands of idle connections.
4. Inspect `/proc/interrupts` before and during a network load test and explain CPU distribution.
5. Trigger a controlled service failure, then reconstruct the sequence from journal monotonic timestamps.

## 1.14 Principal-level summary

> Linux performance is queueing across privilege and subsystem boundaries. I start from customer impact, trace the operation through userspace and the syscall boundary, identify where tasks block or queue, and validate the hypothesis with kernel counters and profiles. I avoid random tuning because the same symptom—high latency—can originate in scheduler delay, reclaim, lock contention, block I/O, networking, firmware, or a dependency outside the host.
