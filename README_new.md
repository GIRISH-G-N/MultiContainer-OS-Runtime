# MultiContainer-OS-Runtime

A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor.

---

## 1. Team Information

| Name | SRN |
|------|-----|
| [Your Name 1] | [SRN 1] |
| [Your Name 2] | [SRN 2] |

---

## 2. Build, Load, and Run Instructions

### Prerequisites
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget
```

### Build Everything
```bash
cd boilerplate
make clean
make
```

### Prepare Root Filesystem
```bash
cd boilerplate
mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta
cp cpu_hog memory_hog io_pulse rootfs-alpha/
cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Load Kernel Module
```bash
cd boilerplate
sudo insmod monitor.ko
ls -l /dev/container_monitor
dmesg | tail
```

### Start Supervisor (Terminal 1)
```bash
cd boilerplate
sudo ./engine supervisor ./rootfs-base
```

### Use CLI (Terminal 2)
```bash
cd boilerplate

# Start containers
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 30"
sudo ./engine start beta  ./rootfs-beta  "/io_pulse 20"

# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Run and wait
sudo ./engine run gamma ./rootfs-alpha "/cpu_hog 5"

# Stop
sudo ./engine stop alpha

# Memory limit test
sudo ./engine start memtest ./rootfs-alpha "/memory_hog 4 500" --soft-mib 20 --hard-mib 40
```

### Check Kernel Logs
```bash
dmesg | grep container_monitor
```

### Clean Up
```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C supervisor in Terminal 1
sudo rmmod monitor
make clean
```

---

## 3. Demo with Screenshots

> Add screenshots after running the demo

| # | What it Shows |
|---|--------------|
| 1 | Two containers running under one supervisor |
| 2 | `engine ps` metadata output |
| 3 | Log file contents from `logs/alpha.log` |
| 4 | CLI command and supervisor response |
| 5 | `dmesg` SOFT LIMIT warning |
| 6 | `dmesg` HARD LIMIT kill event |
| 7 | Scheduling experiment results |
| 8 | Clean teardown, no zombies |

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms

Our runtime uses `clone()` with `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` to create isolated containers. Each container gets its own PID namespace (processes see their own PID space), UTS namespace (own hostname), and mount namespace (own mount table). After `clone()`, the child calls `chroot()` into its assigned rootfs and mounts `/proc` so tools like `ps` work inside. The host kernel is still shared — all containers share the same kernel, scheduler, and network stack.

### 4.2 Supervisor and Process Lifecycle

A long-running supervisor is necessary because the OS requires a parent to `wait()` on children — without it, exited children become zombies. The supervisor uses `SIGCHLD` with `waitpid(-1, WNOHANG)` to reap all exited children immediately and update their metadata. A mutex protects the container list from concurrent access by the signal handler, main loop, and logging threads.

### 4.3 IPC, Threads, and Synchronization

Two IPC paths are used:
- **Logging (Path A):** Pipes from each container's stdout/stderr into the supervisor. Log reader threads push chunks into the bounded buffer.
- **Control (Path B):** UNIX domain socket at `/tmp/mini_runtime.sock` for CLI-to-supervisor communication.

The bounded buffer uses a `pthread_mutex` and two `pthread_cond` variables (`not_empty`, `not_full`). We chose mutex over spinlock because producers/consumers block for extended periods, and spinning would waste CPU. Condition variables also allow clean broadcast-shutdown signaling.

### 4.4 Memory Management and Enforcement

RSS (Resident Set Size) measures physical RAM pages currently present in memory. It does not include swapped pages or unfaulted mappings. Soft limits warn without killing — useful for alerting operators before a problem escalates. Hard limits kill immediately to protect the host and other containers. Enforcement belongs in kernel space because: (1) a runaway process cannot interfere with kernel timers, (2) the kernel has direct access to `mm_struct` without polling overhead, and (3) `SIGKILL` from the kernel cannot be blocked by the container.

### 4.5 Scheduling Behavior

Linux CFS (Completely Fair Scheduler) uses virtual runtime (`vruntime`) to decide which task runs next. Our experiments showed equal CPU share for equal-nice containers, immediate responsiveness for I/O-bound tasks (because sleeping resets their `vruntime` advantage), and measurable CPU share reduction for containers with higher nice values, consistent with CFS weight calculations.

---

## 5. Design Decisions and Tradeoffs

| Subsystem | Decision | Tradeoff | Justification |
|-----------|----------|----------|---------------|
| Namespace isolation | `chroot` + PID/UTS/mount namespaces | `chroot` escapable by root; `pivot_root` more secure | Simpler for lab scope; sufficient isolation for experiments |
| Supervisor architecture | Single-threaded `select()` loop | Cannot handle many concurrent clients | Adequate for demo scale; much simpler than multi-threaded accept |
| IPC control channel | UNIX domain socket | Requires supervisor running first | Natural fit for synchronous request/response CLI |
| Bounded buffer | Fixed circular array (16 slots) | Can block producers if logger is slow | Bounded memory; prevents unbounded log accumulation |
| Kernel lock | `mutex` | Higher overhead than spinlock | Timer callback calls `kmalloc` which can sleep — spinlock would panic |

---

## 6. Scheduler Experiment Results

### Experiment 1: Two CPU-bound containers, equal nice=0

Both `cpu_hog` containers completed in approximately equal time (~20s each), with roughly 50/50 CPU split. CFS distributed CPU fairly between equal-weight tasks.

### Experiment 2: CPU-bound vs I/O-bound (equal priority)

`cpu_hog` saturated the CPU while `io_pulse` remained responsive throughout. CFS rewarded the I/O-bound task's voluntary sleeping by scheduling it immediately on wake-up — demonstrating Linux's natural latency-sensitivity for I/O workloads.

### Experiment 3: Different nice values (nice=0 vs nice=10)

The nice=0 container completed ~2x faster than the nice=10 container. CFS weight (derived from nice value) directly controls CPU share allocation — confirming that `nice` is a meaningful scheduling hint in Linux CFS.
