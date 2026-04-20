# MultiContainer-OS-Runtime

A lightweight Linux container runtime in C with a long-running supervisor and a kernel-space memory monitor — built from scratch as part of the Operating Systems course (UE24CS242B).

---

## Team Information

| Name | SRN |
|------|-----|
| GIRISH G N | PES1UG24CS571 |
| ASHITH RAO K | PES1UG24CS559 |

---

## What is this Project?

This project implements a mini Docker-like container runtime entirely in C. It has two major components:

- **`engine.c`** — A user-space program that creates and manages isolated Linux containers using `clone()` and namespaces. It runs as a long-lived supervisor daemon and accepts CLI commands over a UNIX domain socket.
- **`monitor.c`** — A Linux kernel module that tracks the RAM usage of each running container every second, logs a warning when the soft limit is exceeded, and kills the process when the hard limit is exceeded.

---

## Project Architecture

```
CLI (engine ps / start / stop / logs)
           |
           |  UNIX domain socket (/tmp/mini_runtime.sock)
           v
       SUPERVISOR (engine supervisor)
           |
           |-- clone(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS)
           |       |
           |       v
           |   CONTAINER PROCESS
           |       - chroot() into rootfs
           |       - mount /proc
           |       - exec command
           |
           |-- ioctl(MONITOR_REGISTER)
           |       |
           |       v
           |   KERNEL MODULE (monitor.ko)
           |       - linked list of containers
           |       - timer fires every 1 second
           |       - checks RSS (RAM usage)
           |       - soft limit -> WARN in dmesg
           |       - hard limit -> SIGKILL
           |
           |-- pipe -> log reader thread
                   -> bounded buffer (16 slots)
                   -> logging thread
                   -> logs/<id>.log
```

---

## File Structure

```
MultiContainer-OS-Runtime/
├── boilerplate/
│   ├── engine.c              # User-space supervisor and CLI
│   ├── monitor.c             # Linux kernel module
│   ├── monitor_ioctl.h       # Shared ioctl definitions
│   ├── cpu_hog.c             # CPU-bound test workload
│   ├── io_pulse.c            # I/O-bound test workload
│   ├── memory_hog.c          # Memory-consuming test workload
│   ├── Makefile              # Build system
│   └── environment-check.sh  # VM preflight check
├── project-guide.md
└── README.md
```

---

## Build and Run Instructions

### Prerequisites

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r) wget
```

### Step 1: Build Everything

```bash
cd ~/projects/MultiContainer-OS-Runtime/boilerplate
make clean && make
```

This compiles:
- `engine` — the main runtime binary
- `monitor.ko` — the kernel module
- `cpu_hog`, `io_pulse`, `memory_hog` — test workloads

### Step 2: Verify Environment

```bash
sudo ./environment-check.sh
```

All checks should show `[OK]`.

### Step 3: Prepare Root Filesystem

```bash
cd ~/projects/MultiContainer-OS-Runtime/boilerplate

mkdir -p rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta

cp cpu_hog memory_hog io_pulse rootfs-alpha/
cp cpu_hog memory_hog io_pulse rootfs-beta/
```

### Step 4: Load Kernel Module

```bash
sudo insmod monitor.ko
sudo dmesg | tail -5
```

Expected output:
```
[container_monitor] Module loaded. Device: /dev/container_monitor
```

### Step 5: Start the Supervisor — Terminal 1

```bash
sudo ./engine supervisor ./rootfs-base
```

Expected output:
```
Supervisor started. base-rootfs=./rootfs-base socket=/tmp/mini_runtime.sock
```

Keep this terminal open! The supervisor waits for CLI commands.

### Step 6: Use the CLI — Terminal 2

```bash
cd ~/projects/MultiContainer-OS-Runtime/boilerplate

# Start a CPU-bound container
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 30"

# Start an I/O-bound container
sudo ./engine start beta ./rootfs-beta "/io_pulse 20"

# List all containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha
sudo cat logs/alpha.log

# Check kernel memory monitor
sudo dmesg | grep container_monitor

# Stop containers
sudo ./engine stop alpha
sudo ./engine stop beta
sudo ./engine ps
```

---

## Demo Scenarios

### Demo 1: Basic Container Lifecycle

```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 30"
sudo ./engine ps
# alpha | PID | running | timestamp
sudo ./engine stop alpha
sudo ./engine ps
# alpha | PID | stopped | timestamp
```

### Demo 2: Multiple Containers

```bash
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 30"
sudo ./engine start beta  ./rootfs-beta  "/io_pulse 20"
sudo ./engine ps
sudo dmesg | grep container_monitor
# Both containers registered in kernel with soft/hard limits
```

### Demo 3: Memory Limit Enforcement

```bash
# memory_hog allocates 4MB every 500ms
# soft limit = 20MB (warn), hard limit = 40MB (kill)
sudo ./engine start memtest ./rootfs-alpha "/memory_hog 4 500" --soft-mib 20 --hard-mib 40

# Wait 15 seconds then:
sudo dmesg | grep container_monitor
```

Expected:
```
[container_monitor] SOFT LIMIT container=memtest pid=XXXX rss=21000000 limit=20971520
[container_monitor] HARD LIMIT container=memtest pid=XXXX rss=41000000 limit=41943040
```

### Demo 4: Scheduling Experiment

```bash
sudo ./engine start fast ./rootfs-alpha "/cpu_hog 20" --nice 0
sudo ./engine start slow ./rootfs-beta  "/cpu_hog 20" --nice 10
# fast finishes ~2x sooner than slow
```

### Demo 5: Clean Teardown

```bash
sudo ./engine stop alpha
sudo ./engine stop beta
# Ctrl+C supervisor in Terminal 1
sudo rmmod monitor
sudo dmesg | tail -3
# [container_monitor] Module unloaded.
```

---

## Engineering Analysis

### Isolation Mechanisms

Each container is created using `clone()` with three namespace flags:

| Flag | Namespace | Effect |
|------|-----------|--------|
| `CLONE_NEWPID` | PID | Container sees its own PIDs starting from 1 |
| `CLONE_NEWUTS` | UTS | Container has its own hostname |
| `CLONE_NEWNS` | Mount | Container has its own mount table |

After `clone()`, the child process calls `chroot()` into its assigned rootfs, mounts `/proc`, sets its nice value, then calls `execv()` to run the configured command. The host kernel is still shared — all containers share the same kernel, scheduler, and network stack.

### Supervisor and Process Lifecycle

The supervisor is a long-running parent process necessary because the OS requires a parent to `wait()` on children — without it, dead containers become zombies. It uses a `SIGCHLD` handler with `waitpid(-1, WNOHANG)` to reap all exited children immediately. A mutex protects the container metadata list from concurrent access by the signal handler, main loop, and logging threads.

### IPC and Logging

Two IPC channels are used:

**Path A — Logging:**
Container stdout/stderr → pipe → log reader thread → bounded buffer → logging thread → `logs/<id>.log`

**Path B — Control:**
CLI → UNIX domain socket → supervisor select() loop → response back to CLI

The bounded buffer uses `pthread_mutex` and two `pthread_cond` variables (`not_empty`, `not_full`). We chose mutex over spinlock because log operations are I/O bound and can block for extended periods. Condition variables allow clean broadcast-shutdown signaling on exit.

### Kernel Memory Monitor

Every second, the timer callback iterates the monitored container list:
- Removes entries for processes that no longer exist
- Logs a WARNING if RSS exceeds soft limit (once per container)
- Sends `SIGKILL` and removes entry if RSS exceeds hard limit

**Why kernel space?** A runaway process cannot interfere with kernel timers. The kernel has direct access to `mm_struct` without polling overhead. `SIGKILL` from kernel space cannot be blocked by the container.

**Why mutex over spinlock?** The timer callback calls `kmalloc()` which can sleep. Spinlocks cannot be held while sleeping — this would cause a kernel panic.

### Scheduler Experiments

| Experiment | Setup | Result | Why |
|-----------|-------|--------|-----|
| Equal CPU workloads | Two cpu_hog at nice=0 | ~50/50 CPU split | CFS distributes fairly |
| CPU vs I/O | cpu_hog vs io_pulse at nice=0 | io_pulse stays responsive | Sleeping tasks get lower vruntime, scheduled first on wake |
| Different nice | nice=0 vs nice=10 cpu_hog | nice=0 finishes 2x faster | Nice maps to CFS weight, higher nice = less CPU |

---

## Design Decisions

| Subsystem | Decision | Tradeoff | Justification |
|-----------|----------|----------|---------------|
| Isolation | chroot + PID/UTS/Mount namespaces | chroot escapable by root | Simpler for lab scope |
| Supervisor | Single-threaded select() loop | Cannot handle many concurrent clients | Adequate for demo scale |
| IPC control | UNIX domain socket | Requires supervisor running first | Natural for request/response CLI |
| Bounded buffer | Fixed 16-slot circular array | Can block if logger is slow | Bounded memory, no unbounded accumulation |
| Kernel lock | mutex | Higher overhead than spinlock | kmalloc can sleep, spinlock would panic |

---

## CLI Reference

| Command | Description |
|---------|-------------|
| `./engine supervisor <rootfs>` | Start the supervisor daemon |
| `./engine start <id> <rootfs> "<cmd>" [--soft-mib N] [--hard-mib N] [--nice N]` | Start container in background |
| `./engine run <id> <rootfs> "<cmd>"` | Start container and wait to finish |
| `./engine ps` | List all containers with state |
| `./engine logs <id>` | Show log file for container |
| `./engine stop <id>` | Stop a running container |

---

## GitHub Repository

https://github.com/GIRISH-G-N/MultiContainer-OS-Runtime
