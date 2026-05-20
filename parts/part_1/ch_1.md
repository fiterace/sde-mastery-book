## Chapter 1: OS Fundamentals for Engineers

On the evening of July 2nd, 2019, Cloudflare took down roughly 15% of the internet with a single regex. Every CPU in every Cloudflare point-of-presence globally hit 100% utilization within seconds of a WAF rule deployment. The HTTP proxy — the entire reason customers were using Cloudflare — couldn't process a single request. The regex wasn't talking to any database. It wasn't doing network I/O. It was just spinning in a tight CPU loop, and the operating system's scheduler had no mechanism to help: every runnable thread was doing the same broken work.

The incident is covered in the war stories section. But the reason it belongs at the opening of a chapter on OS fundamentals is this: understanding what happened requires knowing how the Linux CFS scheduler allocates CPU time, what "CPU saturation" actually means versus "high CPU utilization," and why a purely user-space bug can render a server completely unresponsive. Most engineers know the symptoms of OS-level problems. This chapter is about knowing the mechanisms well enough to reason about why.

> **🔖 War story:** Cloudflare's 2019 regex incident is examined in detail in the "How It Breaks" section. The Discord GC pause story is there too.

---

### A. Concept

#### Processes vs Threads

A process is an isolated execution context with its own virtual address space, file descriptor table, and signal handlers. The Linux kernel tracks each process via a `task_struct` — a data structure holding everything from the PID to the memory map to the saved CPU register state.

A thread is a unit of execution within a process. Threads share the same virtual address space and file descriptor table as their parent process. This sharing is what makes threads cheap to create (no new address space mapping needed) and what makes them dangerous (shared state means shared bugs).

The distinction matters in three concrete ways:

1. **Crash isolation** — a segfault in one process cannot reach another process's memory. A segfault in one thread kills the entire process and all its sibling threads.
2. **Communication cost** — threads communicate via shared memory (fast, no serialization, no syscall). Processes communicate via IPC — pipes, sockets, or explicitly shared memory segments (slower, safer, requires explicit synchronization).
3. **Scheduling granularity** — the Linux scheduler schedules threads (internally called "tasks"), not processes. A process with 8 threads can occupy 8 CPU cores simultaneously.

```
Virtual Address Space (per process)
┌─────────────────────────────────────┐
│  Kernel space (same for all procs)  │  0xFFFF...
├─────────────────────────────────────┤
│  Stack (grows ↓)                    │
│    Thread 1 stack                   │
│    Thread 2 stack (separate region) │
├─────────────────────────────────────┤
│  Memory-mapped files / shared libs  │
├─────────────────────────────────────┤
│  Heap (grows ↑)                     │
│    malloc() lives here              │
│    SHARED between all threads       │
├─────────────────────────────────────┤
│  BSS  (uninitialised globals)       │
│  Data (initialised globals)         │
│  Text (executable code, read-only)  │  0x0000...
└─────────────────────────────────────┘
```

#### Additional Diagrams

#### Process Lifecycle State Machine

```
         fork()
new ──────────────> ready
                      │  ↑
              schedule│  │preempt / quantum expires
                      ↓  │
                   running ──── syscall / I/O wait ──> blocked
                      │                                    │
                      │ exit()                  I/O done  │
                      ↓                                    │
                  terminated                          ↑____│
                                                    ready
```

Each state is visible in `/proc/<pid>/status` as a single letter:
- `R` — running or runnable (in the CFS run queue)
- `S` — interruptible sleep (waiting for an event, can be killed)
- `D` — uninterruptible sleep (blocked on disk I/O — cannot be killed with SIGTERM or SIGKILL)
- `Z` — zombie (exited but parent hasn't called `wait()`)
- `T` — stopped (SIGSTOP received)

> **⚠ Gotcha:** A process in `D` state cannot be killed with `kill -9`. The only options are to wait for the I/O to complete or reboot. This state is most common with stale NFS mounts.

#### epoll vs select — Why Connection Count Matters

```
select() / poll() — O(n) per call
──────────────────────────────────
Kernel scans ALL fds each call:

[fd1][fd2][fd3]...[fd10000]  ← scan all 10,000 every time
   0    0    0        1        even though only one is ready

epoll — O(1) for event delivery
──────────────────────────────────
One-time registration:
  epoll_ctl(epfd, EPOLL_CTL_ADD, fd, events)

Kernel installs a callback: when fd becomes ready,
it is pushed into epoll's ready list.

epoll_wait() returns ONLY ready fds:
  [fd10000]  ← one entry, no scanning

At 100,000 open connections:
  select / poll: 100,000 checks per wakeup
  epoll:         1 check per wakeup
```

This is why Nginx, Node.js's event loop, and Go's netpoller all use epoll on Linux.

#### Context Switching

Context switching is the CPU's act of saving one task's register state and loading another's. On x86-64, this means saving and restoring roughly 168 bytes of general-purpose registers plus FPU/SIMD state (potentially 512+ bytes with AVX registers in use).

The direct cost of the register swap: 1–2 microseconds. The real cost is **cache pollution**. When the CPU switches from task A to task B, the L1 and L2 caches still contain A's working set. Task B generates cache misses until its own working set warms up. On a server with 32KB L1 and 256KB L2, a complete eviction costs 1–10 microseconds of effective lost work.

```
CPU Timeline
────────────────────────────────────────────────────────────
Process A  ████████░░░░░░░░████████░░░░░░░░
Process B  ░░░░░░░░████████░░░░░░░░████████
           ↑               ↑
           context switch  context switch
           (save A state,  (save B state,
            load B state)   load A state)

████ = running   ░ = waiting
```

**Voluntary vs involuntary context switches:**
- **Voluntary:** the task blocks — waiting on I/O, a mutex, or calling `sleep()`. The OS takes the CPU and gives it to someone else.
- **Involuntary:** the task's time quantum expires and CFS preempts it.

You can observe them in production:

```bash
# Per-process context switch counts
cat /proc/<pid>/status | grep ctxt

# System-wide context switch rate (cs column)
vmstat 1

# Attach to a running process and count over 5 seconds
perf stat -e cs -p <pid> sleep 5
```

High involuntary context switches (more than ~1,000/second per thread) indicate CPU saturation — more runnable threads exist than available CPU cores.

#### Linux CFS Scheduler

The Completely Fair Scheduler (CFS), introduced in Linux 2.6.23, tracks each task's `vruntime`: the amount of CPU time the task has received, weighted by its nice priority. CFS always runs the task with the lowest `vruntime` — the one that has received the least CPU time relative to its entitlement.

CFS doesn't use fixed time slices. It divides a configurable scheduling period (default: 6ms, or 4ms per CPU when more than 8 tasks are runnable) among all runnable tasks proportionally. A process at `nice 0` gets twice the CPU time of a process at `nice 10`.

```
CFS Red-Black Tree (sorted by vruntime)
           [vruntime=100]
          /               \
    [vruntime=80]    [vruntime=140]
     /       \
  [60]      [90]
   ↑
   Next to run (leftmost node = minimum vruntime)
```

What this means in practice:
- **I/O-bound tasks** accumulate less vruntime while waiting. When they become runnable, they have a lower vruntime than CPU-bound tasks and get scheduled first. This is intentional — it makes interactive/I/O workloads feel responsive.
- **Nice values are meaningful.** Running a batch job at `nice 19` gives it roughly 5% of CPU when competing with a `nice 0` service.
- **NUMA effects.** CFS tries to keep tasks on the same NUMA node for cache locality, but cross-node load balancing incurs 100–300ns extra per memory access on remote memory. For latency-sensitive services on multi-socket servers, CPU affinity pinning is worth measuring.

#### Virtual Memory and Page Faults

Every process sees a private 64-bit virtual address space. The OS and CPU cooperate to translate virtual addresses to physical RAM addresses using **page tables** — a 4-level tree on x86-64, where each leaf entry maps a 4KB virtual page to a physical frame number.

The TLB (Translation Lookaside Buffer) caches recently used page table entries. A TLB hit: ~1 ns. A TLB miss forces a full page table walk: ~100–200 ns. On a service that performs millions of allocations touching many distinct memory regions, TLB pressure becomes measurable as latency.

```
Virtual Address Translation
─────────────────────────────────────────────────────────
  Virtual Address (48 bits used on x86-64)
  ┌──────┬──────┬──────┬──────┬──────────────┐
  │ PML4 │  PDP │  PDE │  PTE │  Page Offset │
  │ idx  │  idx │  idx │  idx │   (12 bits)  │
  └──────┴──────┴──────┴──────┴──────────────┘
      │       │       │       │
   CR3 → PML4 → PDP → PD → PT → Physical Frame + Offset
      └── TLB cache short-circuits all 4 walks on hit ──┘
```

**Page fault types:**
- **Minor fault:** the page table entry is valid but the physical page isn't in the TLB, or the mapping exists but hasn't been backed by a physical frame yet. No disk I/O. Cost: ~1–10 μs to allocate a frame and update the page table.
- **Major fault:** the page is not in physical RAM and must be read from disk (or swap). This blocks the thread. On SSD: ~100 μs minimum. On spinning disk: ~5–10 ms. Production services should have zero major faults under steady load — if you see them, you're swapping.

**mmap and demand paging:**

`mmap(MAP_ANONYMOUS)` allocates virtual address space but does NOT allocate physical pages. Pages are allocated on first access. This is why `malloc(1GB)` returns almost instantly — the gigabyte of physical memory is provisioned lazily as you touch pages.

For Redis RDB snapshots: `fork()` creates a child that shares all parent pages as copy-on-write. The first write to any shared page by the parent triggers a page fault, the kernel copies that 4KB page, and the parent gets a private copy. Under heavy write load during a snapshot, Redis can use up to 2x its normal physical memory.

> **⚠ Gotcha:** Size Redis at 60–65% of available RAM if RDB or AOF rewrite is enabled, not 80%. The remaining headroom is consumed by copy-on-write pages during background saves. OOM kills during snapshot windows are a common production surprise.

#### System Calls

A system call transitions the CPU from user mode (ring 3) to kernel mode (ring 0). The transition involves: saving user-space registers, switching to the kernel stack, executing the `syscall` instruction, dispatching through the system call table, doing the work, and returning via `sysret`.

The raw cost on a modern CPU: ~100–300 ns. With Spectre/Meltdown mitigations (KPTI enabled), this rises to ~1–2 μs. At 10,000 syscalls/second per thread — a reasonable rate for network I/O — this is 1–20 ms/second of overhead, not catastrophic. At 1,000,000 syscalls/second (small-buffer reads in a tight loop), it's a real problem.

```python
import os
import time

N = 1_000_000
start = time.perf_counter()
for _ in range(N):
    os.getpid()  # cheap syscall, no I/O
elapsed = time.perf_counter() - start
print(f"Syscall cost: {elapsed/N*1e9:.0f} ns per call")
# Typical: 200–800 ns depending on CPU model and KPTI state
```

```go
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    N := 1_000_000
    start := time.Now()
    for i := 0; i < N; i++ {
        os.Getpid()
    }
    elapsed := time.Since(start)
    fmt.Printf("Syscall cost: %d ns per call\n", elapsed.Nanoseconds()/int64(N))
}
```

**vDSO (virtual dynamic shared object):** Linux maps a small kernel-provided shared library into every process's address space. Calls like `clock_gettime(CLOCK_MONOTONIC)` and `gettimeofday()` are handled entirely in user space via this vDSO — no ring transition, no kernel entry. This is why `CLOCK_MONOTONIC` costs ~20 ns instead of 300 ns. Time-sensitive hot paths should use it.

#### File Descriptors

Every open file, socket, pipe, epoll handle, eventfd, timerfd, or inotify watch is a file descriptor — an integer indexing into the process's per-process file descriptor table. The table entries point to kernel file objects (holding position, flags, and reference count), which in turn point to inodes or socket structures.

```
Process FD Table          Kernel File Table         Inode / Socket
┌──────────────────┐     ┌─────────────────────┐   ┌──────────────┐
│ fd 0  (stdin)    │────>│ file object          │──>│ inode        │
│ fd 1  (stdout)   │────>│ file object          │──>│ inode        │
│ fd 2  (stderr)   │     │ (offset, flags,      │   │              │
│ fd 3  (socket)   │────>│  refcount)           │──>│ socket buf   │
│ fd 4  (epoll)    │────>│ epoll object         │   │              │
└──────────────────┘     └─────────────────────┘   └──────────────┘
```

Two processes can share the same kernel file object after `fork()` — both hold FDs pointing to the same file position. Writing to the same file from both without `O_APPEND` will interleave writes at the same offset, corrupting output. `O_APPEND` makes the seek-and-write atomic.

The default FD limit is 1,024 on many Linux systems. A service handling 100,000 concurrent TCP connections needs this raised to at least 200,000 (each connection uses 1 FD for the socket, plus FDs for thread stacks, log files, and internal pipes). Set it in `/etc/security/limits.conf` or the systemd unit's `LimitNOFILE` directive.

> **⚠ Gotcha:** Container base images often inherit the host's FD limit or default to 1,024. Kubernetes pods running Nginx or any high-connection service need an explicit `LimitNOFILE` in the container's process supervisor configuration, not just in the host's limits.

#### Signals

Signals are asynchronous notifications delivered by the kernel to a process. Key signals for backend engineers:

- `SIGTERM` (15) — polite shutdown request. Your service should catch this and drain in-flight requests before exiting.
- `SIGKILL` (9) — uncatchable by design. The kernel terminates the process immediately with no cleanup.
- `SIGSEGV` (11) — invalid memory access. Default action: core dump and termination.
- `SIGUSR1/SIGUSR2` — user-defined. Nginx uses `SIGUSR1` to reopen log files after rotation without dropping connections.
- `SIGCHLD` — delivered to a parent when a child process changes state (exits, stops, continues). Not handling this causes zombie accumulation.

Signal delivery is asynchronous — the handler interrupts whatever the process was doing. Signal handlers must only call async-signal-safe functions. The POSIX list is short: `write()`, `_exit()`, arithmetic operations, and a handful of others. `malloc()`, `printf()`, `pthread_mutex_lock()` — none of these are async-signal-safe. Calling them from a signal handler is undefined behavior.

The practical pattern in Python and Go is to set a flag or write a single byte to a pipe in the signal handler, then check the flag or read the pipe in the main loop.

#### /proc Filesystem

`/proc` is a virtual filesystem the kernel provides as a window into its own state. Essential paths for production engineers:

```bash
/proc/<pid>/status     # RSS, virtual size, context switches, state
/proc/<pid>/maps       # all virtual memory mappings with permissions
/proc/<pid>/fd/        # symlinks to every open file descriptor
/proc/<pid>/stat       # raw CPU time, I/O counters (machine-readable)
/proc/net/tcp          # all TCP sockets in the system
/proc/sys/kernel/pid_max
/proc/sys/net/core/somaxconn   # listen backlog limit
/proc/sys/vm/swappiness        # kernel's eagerness to swap
```

```python
def get_process_rss_mb(pid: int) -> float:
    """Read resident set size from /proc without shelling out."""
    with open(f"/proc/{pid}/status") as f:
        for line in f:
            if line.startswith("VmRSS:"):
                kb = int(line.split()[1])
                return kb / 1024
    return 0.0
```

Reading `/proc/self/status` in a signal-safe context is not safe (it calls `open()`, which is not async-signal-safe), but reading it from a background health-check thread is perfectly fine and costs roughly one `open()` + a few hundred bytes of `read()` — microseconds.

#### The Mental Model

Think of the OS as a hotel concierge running a building with more registered guests than actual rooms. Each **process** is a guest with their own suite (virtual address space) and room key (file descriptors). **Threads** are members of the same traveling party — they share the suite but can move around independently. The **scheduler** (CFS) is the concierge deciding whose request to handle next, tracking fairness by how much service each guest has already received. When a guest needs something from the building's infrastructure (network, disk), they call the front desk — that call takes time (syscall cost). The hotel has a finite number of actual rooms (physical RAM); when demand exceeds supply, the concierge uses off-site storage (swap) — dramatically slower. Some guests have a VIP pass (vDSO) that lets them check the time without calling the front desk at all.

#### Common Misconceptions

**1. "Threads are always faster than processes."** Not for CPU-bound Python — the GIL serializes threads regardless of how many CPU cores you have. Use `multiprocessing` for CPU-bound Python work. In Go and Rust, goroutines/threads do truly parallelize, but "faster to create" doesn't mean "faster to execute" — thread scheduling and cache pollution still apply.

**2. "fork() is expensive."** The `fork()` syscall itself is cheap (~50 μs on Linux) because of copy-on-write. The pages are not copied at fork time. The cost is deferred to first write on each shared page. This is why Redis's RDB snapshot model works: the fork is instantaneous; the cost arrives as a tax on subsequent writes.

**3. "A process crash can't affect other processes."** True for private memory. False for shared resources: a process that corrupts a shared file, exhausts the system-wide FD table, or creates millions of zombie processes that fill the PID namespace does affect others.

**4. "Signals are reliable for IPC."** Standard Unix signals (SIGTERM, SIGUSR1, etc.) do not queue beyond one pending delivery. If you send SIGUSR1 twice before the handler runs, the second may be silently dropped. Use a pipe or domain socket for reliable one-to-many notifications.

**5. "More RAM always means better performance."** Once the working set fits in RAM, adding more DRAM changes nothing. Performance is often limited by the cache hierarchy (L1 is ~100 GB/s; DRAM is ~50 GB/s; but L1 latency is ~1 ns vs DRAM's ~60 ns). Services that don't fit in L3 cache — even with 128 GB of RAM — pay DRAM latency on every cache miss.

---

### B. How It Breaks in Production

Now that the mechanisms are clear, here is what they look like when they fail.

#### Discord — 2017–2020

**What happened:** Discord's Go service for reading message history experienced perfectly regular latency spikes: every ~2 minutes, p99 latency jumped from ~1ms to 10+ seconds. The regularity made it diagnosable.

**Root cause:** Go's garbage collector was scanning the entire LRU cache (millions of live Go objects) on each GC cycle. The 2-minute period matched the GC trigger interval under their allocation pattern. During GC stop-the-world pauses, all goroutines were halted. The OS scheduler saw thousands of goroutines become simultaneously runnable after each pause, generating a burst of involuntary context switches and cache pressure that amplified the GC pause duration.

**How they fixed it:** Tuning `GOGC` (Go's GC aggressiveness knob) reduced frequency but not severity. Discord ultimately rewrote the service in Rust, where there is no GC. The Rust version maintained consistent sub-1ms latency with no periodic spikes.

**What to take away:** GC pauses interact with OS scheduling. When a GC event simultaneously unblocks thousands of goroutines, the scheduler must assign CPU time to all of them — this produces a burst of involuntary context switches and cache evictions that compound the GC pause. Language-level tuning can reduce the frequency; it cannot eliminate the mechanism.

**Source:** [Why Discord is switching from Go to Rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust) — Discord Engineering Blog, 2020

---

#### Cloudflare — July 2, 2019

**What happened:** A WAF (Web Application Firewall) rule deployment contained a regex with catastrophic backtracking. Within seconds of deployment, CPU utilization at every Cloudflare point-of-presence globally hit 100%. The HTTP reverse proxy became unable to process requests. Roughly 15% of the internet was unreachable for approximately 27 minutes.

**Root cause:** The regex pattern involved repeated sub-expressions of the form `.*.*` applied to attacker-controlled input. The regex engine's backtracking algorithm is a CPU-bound tight loop — no I/O, no blocking, no syscalls. With every CPU at 100% in user space, the CFS scheduler could context-switch among the regex-running threads, but every thread was doing the same useless work. There was no healthy thread to schedule.

**How they fixed it:** Immediate fix: roll back the WAF rule (1-button deploy rollback). Long-term: added a pre-deployment step that simulates regex execution against adversarial input and rejects any pattern with super-linear worst-case time. Moved new WAF rules to RE2, which guarantees linear-time execution.

**What to take away:** CPU saturation differs from high CPU utilization. At 70% CPU, the OS can preempt busy threads and run others. At 100% CPU with no runnable healthy thread, preemption achieves nothing — every task the scheduler runs is equally broken. Algorithmic complexity attacks (ReDoS, hash flooding) bypass all throughput-based rate limiting because they consume CPU, not connections.

**Source:** [Details of the Cloudflare outage on July 2, 2019](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/) — Cloudflare Blog, 2019

---

### C. Decision Framework

The mechanisms above lead to concrete decision points. Here are the two most common OS-level forks in the road.

```
Choosing between processes and threads for concurrent work:

Is the work CPU-bound?
    │
    ├─ Yes + Python → Use multiprocessing.Process
    │                  (GIL prevents true CPU parallelism in threads)
    │
    ├─ Yes + Go / Rust / Java → threads or goroutines are fine
    │   (no GIL; true parallelism available)
    │
    └─ No (I/O-bound: network, database, disk)
           │
           Is connection count > 1,000?
           │
           ├─ Yes → async I/O (asyncio, Node.js, Go)
           │         epoll-based; one thread handles thousands of connections
           │
           └─ No → thread pool is fine (< 100 threads usually sufficient)
                    │
                    Is crash isolation required?
                    │
                    ├─ Yes → separate processes with supervisord / systemd
                    │        (plugin sandbox, untrusted code, security boundary)
                    └─ No → threads within the same process are sufficient
```

```
Diagnosing high latency from OS-level causes:

Service p99 latency has increased — where to start:

Step 1: Is CPU idle > 20%?
    │
    ├─ No (CPU saturated):
    │   vmstat: is run-queue 'r' > CPU count?
    │       ├─ Yes → scheduling saturation; too many runnable threads
    │       │         Fix: reduce thread count or scale horizontally
    │       └─ No → single thread spinning (CPU-bound hot path)
    │                 Fix: profile with py-spy / pprof to find the loop
    │
    └─ Yes (CPU has headroom):
        │
        ├─ Check major page faults (/proc/<pid>/status: VmRSS growing?)
        │   Major faults > 0 under load → memory pressure; check free/swap
        │
        ├─ Check involuntary context switches (high → host CPU contention)
        │   Another process on the same host is stealing your CPU
        │
        ├─ Check I/O wait (%wa in vmstat > 5%)
        │   Disk reads on the critical path; check iostat -x
        │
        └─ Check syscall rate (perf stat -e syscalls:sys_enter -p <pid>)
            Millions/sec → tight loop doing small reads or getpid-style calls
            Fix: batch I/O, use larger buffers, use vDSO calls for time
```

---

### D. Effort Estimation Guide

| Task | Solo greenfield | Adding to existing service | Notes |
|------|-----------------|---------------------------|-------|
| Add /proc-based health metrics endpoint | 2–4 hours | 1–2 hours | Reading `/proc/self/status` and `/proc/self/stat` is under 50 lines of Python; parsing is simple but tedious |
| Tune FD limits and OS socket parameters for high-concurrency server | 2–4 hours | 1 hour | Changes in `/etc/security/limits.conf` and `sysctl.conf`; requires load test to validate actual behavior under target connection count |
| Implement graceful SIGTERM handler for in-flight request draining | 2–4 hours | 1–3 hours | Complexity scales with background worker count, connection pool teardown, and whether the service uses async or sync I/O |
| Switch CPU-bound Python service from threading to multiprocessing | 1–3 days | 2–5 days | Must audit all shared state (module-level globals, caches, database connections), replace queues, test carefully for data races in the transition |
| Profile and fix a context-switch hot spot | 4–8 hours | 4–8 hours | Profiling with `perf` is fast; the fix often requires an architectural change (fewer threads, async I/O) rather than a code-level tweak |
| Implement mmap-based IPC between two services | 1–2 days | 2–4 days | Synchronization (producer/consumer coordination), handling crashes mid-write, and recovery from partial writes all add scope |

---

### E. Exercises

**Exercise 1 — Process vs Thread Crash Isolation**

Write a Python script that launches 3 worker processes and 3 worker threads. Have each worker increment a counter 1 million times. Then deliberately cause one worker of each type to crash by raising an unhandled exception. Observe which other workers survive in each case.

Constraints: use `multiprocessing` and `threading` modules. Do not catch the exception in the crashing worker.

Evaluation criteria: a thread crash should terminate the entire process group. A process crash should leave sibling processes running. Explain at the OS level why the isolation behavior differs.

---

**Exercise 2 — Context Switch Measurement**

Run a CPU-bound Python loop in a background thread and in the main thread simultaneously. Measure total wall-clock time versus per-thread CPU time (`time.process_time()`). Record `voluntary_ctxt_switches` and `nonvoluntary_ctxt_switches` from `/proc/self/status` before and after.

Constraints: run for at least 5 seconds. Use `threading.Thread`.

Evaluation criteria: wall time should approximately equal per-thread CPU time (no speedup, because of the GIL). Involuntary switches should be non-zero and reflect CFS preemption. The gap between wall time and summed CPU time quantifies OS scheduling overhead.

---

**Exercise 3 — Page Fault Observation**

Allocate a 512 MB byte array in Python. Before allocation, after allocation, and after touching every 4096th byte, read major and minor fault counts from `/proc/self/status`. Measure the time taken for the first scan (touching every page) versus a second scan of the same memory.

Constraints: run on Linux. Compare results with `mmap.mmap(-1, size)`.

Evaluation criteria: the first scan should be slower (demand paging is resolving faults). The second scan should be faster (pages are resident and TLB is warm). Explain the difference in terms of demand paging and TLB state.

---

**Exercise 4 — File Descriptor Limit**

Write a Python script that opens `/dev/null` in a loop until it receives an `OSError`. Record the FD count at failure by inspecting `/proc/self/fd/`. Then raise the limit using `resource.setrlimit(resource.RLIMIT_NOFILE, ...)` and repeat.

Constraints: use the `resource` module. Do not close the FDs in the loop.

Evaluation criteria: you should observe `OSError: [Errno 24] Too many open files`. After raising the soft limit, you should be able to open more. Document the hard vs soft limit behavior: the soft limit can be raised up to the hard limit by the process itself; exceeding the hard limit requires root.

---

**Exercise 5 — SIGTERM Graceful Shutdown**

Implement a Python HTTP server using `http.server.HTTPServer` that catches `SIGTERM`, stops accepting new connections, waits for any in-flight request to complete (simulate with `time.sleep(2)` inside the handler), and logs "clean shutdown" before exiting.

Constraints: test by sending `kill -TERM <pid>` while a request is in flight. The in-flight request must complete.

Evaluation criteria: `kill -9` should produce no "clean shutdown" message. `kill -15` should produce it. The server must not hang indefinitely — implement a maximum drain timeout of 10 seconds.

---

**Exercise 6 — /proc as a Lightweight APM**

Write a Go program that every second reads its own `/proc/self/status`, parses RSS, VmPeak, voluntary context switches, and involuntary context switches, and logs them to stdout. Spawn a few goroutines that perform allocations. Graph the RSS growth over time with the logged data.

Constraints: parse `/proc/self/status` directly with file I/O — do not use a library.

Evaluation criteria: RSS should grow as allocations accumulate. Context switches should increase when goroutines compete for the scheduler. This is the foundation of any custom metrics sidecar or health endpoint.

---

**Exercise 7 — vDSO vs Syscall Timing**

Benchmark `clock_gettime(CLOCK_MONOTONIC)` versus a non-vDSO syscall (e.g., `getpid()`). In Go, use the `syscall` package to make a direct syscall for comparison. Run 10 million iterations of each.

Constraints: measure nanoseconds per call. Run on bare metal or a VM with KPTI visible behavior.

Evaluation criteria: `clock_gettime` via vDSO should be 5–20x faster than a genuine ring-transition syscall. If the values are similar, the kernel on that VM may have vDSO disabled or KPTI overhead is unusually low. Note the environment.

---

### F. Problem Set

**Problem 1 — Zombie Reaping [Easy]**

A deployment script forks a child process to run a cleanup job but never calls `wait()`. After the child exits, what happens to its process table entry? How many zombies can accumulate before the system cannot fork new processes?

Write code using three different patterns to avoid zombies.

*Reference solution:*

```python
import os
import signal

# Pattern 1: explicit waitpid
pid = os.fork()
if pid == 0:
    # child work
    os._exit(0)
os.waitpid(pid, 0)  # blocks until child exits; reaps it

# Pattern 2: double-fork (orphan to init, PID 1 reaps automatically)
pid = os.fork()
if pid == 0:
    if os.fork() > 0:
        os._exit(0)      # first child exits immediately
    # grandchild runs here; reparented to init
    do_background_work()
    os._exit(0)
os.waitpid(pid, 0)       # immediately reaps the first child

# Pattern 3: SIGCHLD handler with WNOHANG
def reap_children(signum, frame):
    while True:
        try:
            os.waitpid(-1, os.WNOHANG)
        except ChildProcessError:
            break

signal.signal(signal.SIGCHLD, reap_children)
```

The PID namespace defaults to 32,768 on 32-bit systems and up to ~4 million on 64-bit. Each zombie holds one PID and one kernel `task_struct`. When the table is full, no new processes can be forked — including health checks and shell commands, which is why zombie accumulation is a production-impacting bug.

---

**Problem 2 — Context Switch Rate Under Load [Medium]**

A Python web service uses a thread pool of 500 threads. Under moderate load (200 req/s), p99 latency is 200ms. Under high load (800 req/s), all threads are busy and p99 spikes to 5 seconds. The application logic handles database queries (average: 15ms each). The host has 16 CPU cores.

Diagnose the cause and propose a fix with expected latency improvement.

*Reference solution:*

With 500 threads runnable on 16 cores, CFS allocates each thread approximately 6ms / 31 ≈ 0.2ms of CPU per scheduling cycle. A request requiring 15ms of I/O wait plus 5ms of CPU time across 31 threads scheduled round-robin waits through many scheduling cycles just to accumulate its 5ms of CPU time. The 5-second p99 is consistent with deep queueing in the scheduler combined with cache thrashing on every context switch.

Fix: reduce the thread count to match realistic concurrency. For I/O-bound handlers with 15ms average wait time at 800 req/s: Little's Law gives concurrency = arrival_rate × wait_time = 800 × 0.015 = 12 threads in flight. A pool of 32–64 threads is more than sufficient. The reduction from 500 to 64 threads reduces involuntary context switches by ~8x and eliminates scheduler saturation.

Alternatively, switch to asyncio-based async handlers — a single thread handles thousands of concurrent I/O-bound requests via epoll without context switching.

---

**Problem 3 — File Descriptor Leak [Medium]**

Your service runs cleanly for hours, then dies with "Too many open files." The problem recurs after 6–8 hours. The service processes ~1,000 HTTP requests/minute.

Write the complete diagnosis procedure including shell commands, and show the code pattern that most commonly causes this.

*Reference solution:*

```bash
# Step 1: watch FD count trend over time
watch -n 5 'ls /proc/<pid>/fd | wc -l'

# Step 2: when elevated, inspect what types are leaking
ls -la /proc/<pid>/fd | awk -F'-> ' '{print $2}' | sort | uniq -c | sort -rn

# Step 3: check for sockets stuck in CLOSE_WAIT (kernel code 0x08)
awk '$4 == "08"' /proc/net/tcp | wc -l

# Step 4: check soft FD limit
cat /proc/<pid>/limits | grep files
```

The most common cause in Python services is failing to close HTTP response bodies:

```python
# Bug: exception path skips close(), socket stays open
def call_api(url: str) -> dict:
    r = requests.get(url, timeout=5)
    if r.status_code != 200:
        raise ValueError(f"Bad status: {r.status_code}")
    return r.json()
    # r.close() is never called if status != 200

# Fix: always use the context manager
def call_api(url: str) -> dict:
    with requests.Session() as session:
        with session.get(url, timeout=5) as r:
            r.raise_for_status()
            return r.json()
```

---

**Problem 4 — CoW Memory Cost During Snapshot [Hard]**

A Redis-like service uses fork() to write a background snapshot. The parent is write-heavy at 200 MB/s. The snapshot takes 20 seconds. The service's normal resident memory is 8 GB.

Estimate the peak physical memory required during the snapshot and the performance overhead imposed on the parent by copy-on-write page handling.

*Reference solution:*

Dirty data during snapshot window: 200 MB/s × 20 s = 4 GB of writes. Each write touches at least one 4KB page, so 4 GB / 4 KB = ~1,000,000 page copies. Each copy: kernel handles a page fault, allocates a new physical frame, memcpy 4KB, updates the page table entry.

Peak physical memory: original 8 GB (shared, CoW) + up to 4 GB of copied pages = 12 GB worst case. In practice, writes may cluster, reducing unique pages touched; estimate 10–11 GB.

Performance overhead per copy-on-write fault: ~2–5 μs in the kernel path. At 1,000,000 faults over 20 seconds = 50,000 faults/second = 0.1–0.25 seconds of CPU time spent just in fault handlers, plus increased TLB pressure as page table entries change. Effective throughput reduction: 5–15% for CPU-bound workloads; minimal for I/O-bound ones.

Mitigation: size the host at 1.5× the service's peak RSS when background saves are enabled. Alternatively, use incremental/differential snapshots via dirty-page tracking (`userfaultfd`) to reduce the write rate that triggers CoW.

---

**Problem 5 — Signal Handler Race Condition [Hard / Staff-level]**

A Go HTTP server installs a `SIGTERM` handler that calls `log.Printf()` to record shutdown time and then calls a cleanup function that acquires a `sync.Mutex`. Under load testing, the server occasionally deadlocks on shutdown, hanging forever instead of exiting.

Explain the race condition, identify why it's platform-dependent, and provide a safe alternative pattern.

*Reference solution:*

`log.Printf()` in Go's standard library acquires an internal mutex on the logger. If the goroutine that receives `SIGTERM` runs the signal handler while another goroutine is inside `log.Printf()` holding the logger's lock, the signal handler tries to acquire the same lock — deadlock.

The cleanup function acquiring a `sync.Mutex` has the same problem: if the mutex is held by a goroutine that is waiting for something that won't happen during shutdown (e.g., a queue drain), the signal handler blocks forever.

This is platform-dependent because on Linux, signal delivery goes to an arbitrary thread (goroutine scheduled on an OS thread). On some other platforms, signal masking behavior differs.

Safe alternative in Go:

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
)

func main() {
    srv := &http.Server{Addr: ":8080"}

    // Signal channel pattern: no locks, no logging in handler
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)

    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatalf("server error: %v", err)
        }
    }()

    <-quit  // Block until signal received; no work done in handler

    // Do all cleanup work here, in normal goroutine context
    log.Println("shutdown initiated")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("shutdown error: %v", err)
    }
    log.Println("clean shutdown")
}
```

The signal handler does exactly one thing: send to a buffered channel. All subsequent cleanup runs in normal goroutine context where locks are safe to acquire.

---

### G. Interview Prep

#### Question 1: "Explain the difference between a process and a thread. When would you choose one over the other?"

**Mediocre answer:** A process is an independent program with its own memory. A thread is lighter and shares memory with other threads. Use threads for parallelism.

**SDE-2 answer:** The core distinction is address space isolation. Processes have private virtual address spaces — memory corruption in one process cannot reach another. Threads share the same address space, so they communicate cheaply via shared memory, but a null pointer dereference in one thread kills the entire process.

For the choice: I use threads when tasks need to share large data structures and IPC copy cost would be prohibitive, and when I trust the code's correctness. I use separate processes when I need crash isolation (plugin sandboxing, untrusted code), independent restart behavior, or when the language runtime forces it — Python's GIL means CPU-bound Python threads don't truly parallelize. For a CPU-bound Python service, `multiprocessing.Process` is not a preference, it's a requirement.

**Staff-level answer:** The process/thread choice is really about failure domain size and shared state. At the architectural level, microservices are just processes — you get crash isolation and independent deployment at the cost of serialization and network latency. When a team asks "should we split this into a service?", part of the answer is the same tradeoff you're making when you choose process vs thread.

At scale, there's a NUMA dimension: threads that cross NUMA node boundaries pay 100–300 ns extra per remote memory access. For latency-sensitive services (ad bidding, HFT), CPU affinity pinning to a single NUMA node is the difference between 50μs and 200μs p99. I'd measure with `numastat` before assuming the thread model is the bottleneck.

**Common follow-ups:** How does the Python GIL work? What's a goroutine? How does fork() + exec() work?

**Red flags:** Claiming threads are always faster than processes; not knowing the GIL prevents CPU parallelism in Python; confusing creation cost with execution performance.

---

#### Question 2: "What is a context switch and why does it matter for performance?"

**Mediocre answer:** The OS switches between processes. Too many context switches slow the system down.

**SDE-2 answer:** A context switch saves the current task's CPU registers (168+ bytes on x86-64, up to 512+ with AVX state) and loads another task's. The direct cost is ~1–2 μs. The real cost is cache pollution: the CPU's L1/L2 caches hold the outgoing task's working set, and the incoming task generates cache misses until its own data warms up.

I track context switches in production via `/proc/<pid>/status`. High involuntary switches indicate CPU saturation — more runnable threads than cores. High voluntary switches are usually fine — threads are correctly blocking on I/O rather than spinning.

**Staff-level answer:** Context switches connect to queueing theory. Little's Law says latency = concurrency / throughput. When context switching adds to effective service time, it raises both. CPU pinning prevents the cold-start effect — a pinned thread always returns to the same core and finds its L1/L2 warm. This is why latency-critical services (financial systems, real-time bidding) use CPU isolation (`isolcpus` kernel parameter) to dedicate cores to specific processes.

The Cloudflare outage illustrates the limit of context switching as a remedy: when every CPU is running broken code, preempting one broken thread and running a different broken thread achieves nothing.

**Common follow-ups:** Voluntary vs involuntary switches? How do you detect CPU saturation? What is CPU pinning and when is it worth it?

**Red flags:** Not distinguishing register cost from cache pollution cost; not knowing how to measure context switches; saying context switches are always negligible.

---

#### Question 3: "What happens when your program calls malloc()?"

**Mediocre answer:** malloc allocates memory on the heap and returns a pointer.

**SDE-2 answer:** For allocations under ~128KB, `malloc()` calls into the allocator (glibc's ptmalloc, or jemalloc/tcmalloc if linked). The allocator tries to satisfy the request from its free list — no syscall, just pointer manipulation. If the heap must grow, it calls `brk()` — one syscall. For allocations above ~128KB, malloc calls `mmap(MAP_ANONYMOUS)` directly.

Critically, `mmap()` allocates virtual address space, not physical RAM. Physical pages are allocated on first write via demand paging. `malloc(1GB)` returns almost instantly. The gigabyte of physical pages is provisioned as you touch them. This is why a process can show 4GB of virtual memory and 200MB of RSS — the rest is virtual space that hasn't been backed by physical pages yet.

**Staff-level answer:** The malloc-as-virtual-allocation story has a direct production consequence for Redis and any fork()-based snapshot model: copy-on-write overhead. Every page the parent writes after fork() triggers a new physical frame allocation. At 200MB/s write rate during a 30-second snapshot, that's 6GB of CoW pages — potentially doubling RSS. Teams that size Redis at 80% of RAM without accounting for this get OOM kills during snapshot windows.

Allocator fragmentation is the other angle: long-lived services that allocate and free many differently-sized objects end up with fragmented heaps. jemalloc (used by Redis, Firefox) has significantly better fragmentation characteristics for this workload than glibc's ptmalloc. If RSS grows steadily without a corresponding growth in live objects, compare with jemalloc before declaring a memory leak.

**Common follow-ups:** Stack vs heap? What is a page fault? Why does Redis use extra memory during background saves?

**Red flags:** Not knowing demand paging; claiming malloc always calls the kernel; not knowing Python has its own allocator above malloc.

---

#### Question 4: "You're on call. CPU is at 70% and p99 latency has spiked 5x. What do you check?"

**Mediocre answer:** Check CPU usage, look for a slow endpoint, maybe restart the service.

**SDE-2 answer:** 70% CPU with 5x latency spike suggests the bottleneck is not CPU capacity — something else is the constraint. My steps:

1. `vmstat 1`: check the `r` column (run queue depth). If `r` exceeds the CPU count, scheduling saturation exists even at "70% average" — bursts are hitting 100%.
2. Check major page faults in `/proc/<pid>/status`. Major faults mean disk reads on the request path.
3. Check `%wa` in vmstat. High I/O wait means disk is saturated.
4. For Python: use `py-spy top` to see what threads are waiting on. GIL contention looks like CPU usage without real computation.
5. Check `netstat -s | grep overflow` for TCP accept queue drops. If the service is dropping connections, latency at the application layer is misleading — clients are waiting before they even connect.

**Staff-level answer:** The 70% CPU / 5x latency pattern fits M/M/1 queueing theory exactly. Mean latency = base_latency / (1 - utilization). At 50% utilization, multiplier is 2x. At 70%, it's 3.3x. At 80%, it's 5x. If we calibrated at 40% CPU and are now at 80%, the latency increase is not a bug — it's the system operating as the math predicts. The fix is capacity, not a code change.

If latency is worse than the theoretical model predicts, there's a new bottleneck: lock contention, connection pool exhaustion, or a hot spot introduced by a recent deployment. I'd correlate the latency spike with deployment history before assuming it's traffic growth.

The monitoring I'd add after resolution: per-endpoint p99, CPU saturation (run queue depth, not just utilization %), and a Cloudwatch/Prometheus alarm at 60% CPU utilization rather than 80% — giving 20% headroom before the latency curve steepens.

**Common follow-ups:** CPU utilization vs CPU saturation? How would you distinguish memory pressure from CPU pressure? What monitoring would you add?

**Red flags:** Only looking at average CPU; not knowing vmstat or /proc; not considering queueing theory when "70% CPU" is described as "not the problem."

---

#### Question 5: "What is the /proc filesystem and how do you use it in production?"

**Mediocre answer:** /proc is a virtual filesystem with information about running processes.

**SDE-2 answer:** `/proc` is the kernel's live window into its own state, exposed as a virtual filesystem. For production use, the most valuable paths are:

- `/proc/<pid>/status` — RSS, VmPeak, context switches, process state
- `/proc/<pid>/fd/` — count and type of open file descriptors (leak detection)
- `/proc/<pid>/maps` — all virtual memory mappings (debugging memory layout or finding which library is loaded)
- `/proc/net/tcp` — all TCP sockets including state (detect CLOSE_WAIT accumulation)
- `/proc/sys/net/core/somaxconn` — listen backlog limit (set this to at least 1024 for web servers)

I read `/proc` directly in service health endpoints because it requires no additional tooling, works inside containers, and parsing a few lines of text is microsecond-level overhead.

**Staff-level answer:** `/proc` is also the foundation for most production diagnostic tools: `ps`, `top`, `netstat`, `ss`, `lsof` all read from `/proc`. When those tools aren't available in a minimal container, you can get the same data by reading the files directly. I've written a tiny Go binary that reads `/proc/self/status` + `/proc/net/tcp` on a 30-second interval and pushes to CloudWatch custom metrics — 150 lines of Go, no dependencies, ships in the container image. Much lighter than a full metrics agent.

The `/proc/sys/` subtree is also writable for tuning kernel parameters at runtime without a reboot. `echo 65535 > /proc/sys/net/core/somaxconn` immediately increases the TCP listen backlog. Changes here are ephemeral; persist them in `/etc/sysctl.conf` or a `sysctl -w` call in the instance startup script.

**Common follow-ups:** How would you write a health check that reads /proc? What /proc paths do you check when investigating an OOM kill? How do you persist sysctl settings across reboots?

**Red flags:** Treating /proc as "only for debugging, not production use"; not knowing /proc/net paths for socket inspection; not knowing about /proc/sys for kernel tuning.

---

### Chapter Summary

OS fundamentals are not academic — they are the explanation for a large fraction of the production incidents that don't obviously point to a specific service or query.

**One-sentence summary:** The Linux OS's process model, virtual memory, scheduler, syscall interface, and /proc filesystem are the substrate that every service runs on, and understanding their mechanics is the difference between guessing at latency spikes and diagnosing them in minutes.

**Three things to do this week:**

1. Run `cat /proc/self/status` and `cat /proc/$$/maps` in a terminal on a Linux host (or inside a running container with `docker exec`). Read every line and identify what you know and what you don't. Look up the three fields you don't recognize.

2. Add a `/proc/<pid>/status` reader to a service you own and expose RSS, voluntary context switches, and involuntary context switches as metrics (or as fields in the health check endpoint). Watch what they do under load.

3. Check the `LimitNOFILE` setting in every systemd unit file or container spec you own. If any of them don't explicitly set it, find out what the effective limit is and whether it's appropriate for the number of connections that service handles.

**Cross-references:**

- If this raised questions about how concurrency primitives interact with the OS scheduler, see Chapter 3 (Concurrency and Parallelism).
- If you want to understand how disk I/O and storage work at the layer just below the OS syscall interface, see Chapter 6 (Storage and I/O).
- If you want to apply these ideas to diagnosing a slow service in production, see Chapter 28 (My service is slow).
- If the virtual memory and page table content made you want to understand storage engines, see Chapter 4 (Database Internals — Relational) which covers how Postgres uses the OS memory model.
