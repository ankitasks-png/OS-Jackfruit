# 🌱 OS JACKFRUIT — Multi-Container Runtime

> A lightweight Linux container runtime written in C.

It runs multiple containers at the same time under a single long-running supervisor, logs their output through a bounded-buffer pipeline, exposes a simple CLI to manage them, enforces memory limits using a kernel module we wrote, and includes scheduling experiments to observe how the Linux CFS scheduler behaves under different workloads.

---

## Team

| Name | SRN |
|------|-----|
| Ankita S | PES1UG24CS066 |
| B Nuthan | PES1UG24CS106 |

---

## What We Built

| Component | Description |
|-----------|-------------|
| `engine` | Supervisor + CLI — starts containers, manages lifecycle, routes log output |
| `monitor.ko` | Kernel module that watches per-container memory and enforces soft/hard limits |
| `cpu_hog` | Workload binary: CPU-bound loops for scheduling experiments |
| `memory_hog` | Workload binary: allocates memory to test limit enforcement |
| `io_pulse` | Workload binary: I/O bursts to simulate mixed workloads |
| `run_experiments.sh` | Automates the CFS scheduling experiment suite |

---

## How to Build and Run

### 1. Dependencies

```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### 2. Build

```bash
cd boilerplate
make
```

Produces: `engine`, `cpu_hog`, `memory_hog`, `io_pulse` binaries and the `monitor.ko` kernel module.

### 3. Prepare Root Filesystems

```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base

cp -a rootfs-base rootfs-alpha
cp -a rootfs-base rootfs-beta

cp cpu_hog memory_hog rootfs-alpha/
cp cpu_hog io_pulse   rootfs-beta/
```

### 4. Load the Kernel Module

```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail -10
```

### 5. Start the Supervisor (Terminal 1)

```bash
sudo ./engine supervisor ./rootfs-base
```

### 6. Use the CLI (Terminal 2)

```bash
# Start containers
sudo ./engine start alpha ./rootfs-alpha "/cpu_hog 10"
sudo ./engine start beta  ./rootfs-beta  "/cpu_hog 10"

# Check running containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop a container
sudo ./engine stop alpha
```

### 7. Memory Limit Tests

**Soft limit** — triggers a warning in `dmesg`:
```bash
sudo ./engine start alpha ./rootfs-alpha "/memory_hog 5 30" --soft-mib 1 --hard-mib 512
sleep 5
sudo dmesg | grep container_monitor | tail -10
```

**Hard limit** — container is killed when it exceeds the limit:
```bash
sudo ./engine start alpha ./rootfs-alpha "/memory_hog 10 30" --soft-mib 1 --hard-mib 64
sleep 5
sudo dmesg | grep container_monitor | tail -10
sudo ./engine ps
```

### 8. Scheduling Experiments

```bash
cp -a rootfs-base rootfs-exp1a && cp -a rootfs-base rootfs-exp1b
cp -a rootfs-base rootfs-exp2a && cp -a rootfs-base rootfs-exp2b
cp cpu_hog rootfs-exp1a/ && cp cpu_hog rootfs-exp1b/
cp cpu_hog rootfs-exp2a/ && cp io_pulse rootfs-exp2b/

sudo ./run_experiments.sh
```

### 9. Cleanup

```bash
# Ctrl+C in Terminal 1 to stop the supervisor, then:
ps aux | grep Z | grep -v grep
sudo rmmod monitor
sudo dmesg | grep monitor | tail -5
```

---

## Notes

- The supervisor must be running before issuing any CLI commands.
- All `engine` and `insmod` commands require `sudo`.
- Each `rootfs-*` directory is an independent Alpine Linux filesystem — copy workload binaries into the relevant rootfs **before** starting the container.
- Check `dmesg` for kernel module events, soft-limit warnings, and hard-limit kills.

---

*PES University · 2024*
