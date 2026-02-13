# LINUX & SYSTEMS — FILE 04 of 48
## LINUX PERFORMANCE
### CPU, Memory, Disk I/O, Process Tracing, Kernel Tuning, Benchmarking
**Weeks 7-8 | Mar 29 - Apr 9, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY PERFORMANCE SEPARATES GOOD SREs FROM GREAT ONES**
>
> Any SRE can restart a service. Great SREs find WHY it needed restarting.
> Apple/Google interview: "A server is slow. Walk me through debugging."
> THIS FILE is your answer.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | The USE Method | Utilization, Saturation, Errors framework |
| 2 | CPU Deep Dive | user/system/iowait/steal, load avg, mpstat |
| 3 | Memory Deep Dive | RSS/VSZ, page cache, swap, OOM killer |
| 4 | Disk I/O | IOPS, throughput, latency, iostat, iotop |
| 5 | Practice Lab I | Combined performance investigation |
| 6 | strace & ltrace | System call tracing, debugging hangs |
| 7 | perf & Profiling | CPU profiling, flame graphs |
| 8 | Kernel Tuning | sysctl, /proc/sys, production settings |
| 9 | Benchmarking | stress-ng, fio, sysbench, methodology |
| 10 | Capstone | Full performance investigation scenario |

---

## DAY 1: THE USE METHOD
**Mar 29, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: USE method, first 60 seconds checklist
> 00:20-01:10 Hands-on: run every command, build a USE table
> 01:10-01:30 Drills: interpretation exercises

### The USE Method — By Brendan Gregg

```
For EVERY resource (CPU, Memory, Disk, Network), check THREE things:

  U = UTILIZATION   How busy is it? (% of capacity used)
  S = SATURATION    Is work queuing? (demand beyond capacity)
  E = ERRORS        Any error events?

If U < 70% AND S = 0 AND E = 0 → resource is fine, move on.
If ANY are bad → you found the bottleneck.
```

### USE Applied to Each Resource

```bash
# CPU:
#   Utilization:  top → %us + %sy (< 70%)
#   Saturation:   vmstat → 'r' column (run queue < 2× cores)
#   Errors:       perf stat → cache misses; dmesg | grep mce

# MEMORY:
#   Utilization:  free -h → used/total (watch 'available')
#   Saturation:   vmstat → si/so columns (swap in/out — should be 0!)
#   Errors:       dmesg | grep -i 'oom\|out of memory'

# DISK:
#   Utilization:  iostat -x → %util (< 70% HDD; SSD use await instead)
#   Saturation:   iostat -x → avgqu-sz (queue > 1 = queueing)
#   Errors:       dmesg | grep -i 'error\|fault\|i/o'

# NETWORK:
#   Utilization:  sar -n DEV → rxkB/s, txkB/s vs link speed
#   Saturation:   netstat -s | grep retrans
#   Errors:       ip -s link → errors, drops, overruns
```

### The First 60 Seconds — Brendan Gregg Checklist

```bash
# MEMORIZE THIS. Run these 10 commands first during any incident.

uptime                    # 1. Load averages (> CPU count = overloaded)
dmesg -T | tail -20       # 2. Kernel errors (OOM? disk? hardware?)
vmstat 1 5                # 3. CPU/memory/IO overview
                          #    r=run queue, si/so=swap, us/sy/wa=CPU
mpstat -P ALL 1 3         # 4. Per-CPU (one at 100%? single-thread bottleneck)
pidstat 1 5               # 5. Per-process CPU usage
iostat -xz 1 5            # 6. Disk I/O (await=latency, %util=busy)
free -h                   # 7. Memory ('available' is key, not 'free')
sar -n DEV 1 5            # 8. Network throughput
sar -n TCP 1 5            # 9. TCP rates (active/s, retrans/s)
top                       # 10. Overall picture (1=per-CPU, P=sort CPU)
```

### DAY 1 EXERCISES

> **EXERCISE 1:** Run all 10 checklist commands. Record and interpret results.
> **EXERCISE 2:** Build a USE table: CPU/Mem/Disk/Network × Utilization/Saturation/Errors.
> **EXERCISE 3:** Explain each vmstat column: r, b, swpd, free, si, so, us, sy, wa.
> **EXERCISE 4:** Load average is 8.0 on a 4-core system. Good or bad? Why?
> **EXERCISE 5:** Write a script that runs the 60-second checklist and saves to a file.
> **EXERCISE 6:** Load avg 1-min=12, 5-min=8, 15-min=4. Is the situation improving or worsening?

---

## DAY 2: CPU DEEP DIVE
**Mar 30, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: CPU time breakdown, load average, steal
> 00:20-01:10 Hands-on: stress tests and diagnosis
> 01:10-01:30 Drills: CPU analysis challenges

### CPU Time Categories

```bash
# From top or mpstat:
# %us (user)     Application code              → CPU-bound app
# %sy (system)   Kernel/syscalls               → Too many syscalls (strace)
# %wa (iowait)   Waiting for disk I/O          → Disk problem, NOT CPU!
# %st (steal)    Stolen by hypervisor (VMs)    → Upgrade instance
# %si (softirq)  Network interrupt handling    → High packet rate
# %id (idle)     Nothing to do                 → CPU is fine

# INTERPRETATION:
# High %us       → App is CPU-bound. Profile it. Add cores.
# High %sy       → Lots of syscalls. Use strace to find which.
# High %wa       → I/O bottleneck. Check disk (iostat). NOT a CPU issue!
# High %st       → VM throttled. Bigger instance or dedicated host.
```

### CPU Commands

```bash
mpstat -P ALL 1 5                # per-CPU utilization
# Watch for: one CPU at 100%, others idle = single-threaded bottleneck

pidstat -u 1 5                   # per-process CPU
top -H -p $(pgrep -f myapp)     # per-THREAD CPU within a process
pidstat -w 1 5                   # context switches per process
# cswch/s  = voluntary (waiting for I/O — normal)
# nvcswch/s = involuntary (preempted — system overloaded)

# Load average interpretation:
cat /proc/loadavg
# 4.50 3.20 2.10 3/245 12345
# 1min 5min 15min running/total lastPID
# Rule: load > CPU cores = overloaded
# Compare 1 vs 15: rising = getting worse, falling = recovering
```

> **CPU STEAL — THE CLOUD GOTCHA**
>
> %st means hypervisor gave YOUR CPU time to other VMs.
> App is slow, but local CPU looks low. Common on shared instances.
> Fix: bigger instance, or dedicated/bare-metal host.

### DAY 2 EXERCISES

> **EXERCISE 1:** Run mpstat -P ALL 1. Any single CPU at 100%?
> **EXERCISE 2:** `stress-ng --cpu 2 --timeout 30s` — watch top and mpstat during stress.
> **EXERCISE 3:** `dd if=/dev/zero of=/dev/null bs=1M count=10000` — is this %us or %wa?
> **EXERCISE 4:** pidstat -u 1 5 — what's the top CPU consumer?
> **EXERCISE 5:** Load avg 4.5 on 2 cores. How overloaded? (2.25×)
> **EXERCISE 6:** "App slow but CPU 30%." Possible causes? (iowait, network, single-thread, steal)

---

## DAY 3: MEMORY DEEP DIVE
**Mar 31, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: free vs available, page cache, swap, OOM
> 00:20-01:10 Hands-on: memory stress tests, OOM investigation
> 01:10-01:30 Drills: memory analysis

### Understanding Linux Memory

```bash
free -h
#               total    used    free    shared  buff/cache  available
# Mem:          16Gi     10Gi    500Mi   200Mi   5.5Gi       5.2Gi
# Swap:         8.0Gi    0B      8.0Gi

# CRITICAL: Look at 'available', NOT 'free'.
# 'free'      = truly unused (often tiny — Linux caches aggressively)
# 'available' = free + reclaimable cache = what apps CAN use
# Linux uses ALL free RAM for disk cache. This is GOOD.
# Cache is instantly reclaimable when apps need memory.

# RSS vs VSZ:
# RSS (Resident Set Size) = physical RAM used NOW ← THIS MATTERS
# VSZ (Virtual Size) = total virtual address space (includes mapped files)
# VSZ >> RSS is normal. Don't panic about high VSZ.

ps aux --sort=-%mem | head -10    # top memory consumers
```

### /proc/meminfo Deep Dive

```bash
grep -E "MemTotal|MemAvailable|Buffers|Cached|SwapTotal|SwapFree|Dirty|Slab" /proc/meminfo

# MemTotal      Total physical RAM
# MemAvailable  Free + reclaimable (THE real number)
# Cached        Page cache (file contents cached in RAM)
# Dirty         Pages modified but not yet written to disk
# Slab          Kernel data structure cache
# SwapFree      Free swap (SwapTotal - SwapFree = swap used)
```

### Swap and Swappiness

```bash
# Active swapping = DISASTER (disk is 1000× slower than RAM)

vmstat 1 5
# si = swap in (disk → RAM), so = swap out (RAM → disk)
# BOTH should be 0 in healthy systems!

# Swappiness (0-100): how aggressively kernel swaps
cat /proc/sys/vm/swappiness         # default: 60
sudo sysctl vm.swappiness=10        # better for databases
# Persist: echo "vm.swappiness=10" >> /etc/sysctl.d/99-custom.conf

# Which processes are swapped?
for pid in /proc/[0-9]*; do
    swap=$(awk '/VmSwap/{print $2}' "$pid/status" 2>/dev/null)
    name=$(cat "$pid/comm" 2>/dev/null)
    [[ -n "$swap" && "$swap" -gt 0 ]] && echo "$swap kB $name ($(basename $pid))"
done | sort -rn | head -10
```

### OOM Killer

```bash
# RAM + swap exhausted → kernel invokes OOM Killer
# Picks the process with highest oom_score and kills it.

# Check for OOM kills:
dmesg | grep -i "oom\|killed process\|out of memory"
journalctl -k --grep="oom"

# Protect critical processes:
echo -1000 > /proc/$(pidof postgresql)/oom_score_adj

# INTERVIEW: "App died with no logs." → "Check dmesg for OOM.
#   OOM killer leaves a trace in kernel logs even though
#   the app had no chance to log anything."
```

### DAY 3 EXERCISES

> **EXERCISE 1:** Run `free -h`. What % of RAM is actually available?
> **EXERCISE 2:** Check /proc/meminfo. How much is in page cache?
> **EXERCISE 3:** `stress-ng --vm 2 --vm-bytes 1G --timeout 30s` — watch free and vmstat.
> **EXERCISE 4:** Check vmstat si/so. Any swapping? Which processes?
> **EXERCISE 5:** Set swappiness=10. Why is this better for databases?
> **EXERCISE 6:** Check dmesg for OOM kills. What oom_score does your largest process have?
> **EXERCISE 7:** Process: 2GB RSS, 8GB VSZ. Memory leak? (Probably not — VSZ includes shared libs.)

---

## DAY 4: DISK I/O
**Apr 1, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: IOPS, throughput, latency, HDD vs SSD
> 00:20-01:10 Hands-on: iostat, iotop, disk stress
> 01:10-01:30 Drills: disk analysis

### Disk Performance Metrics

```
              HDD           SSD          NVMe
IOPS          75-200        10K-100K     100K-1M
Throughput    100-200 MB/s  500-3000     2000-7000
Latency       5-15ms        0.1-1ms      0.02-0.1ms
```

### iostat — THE SRE Disk Tool

```bash
iostat -xz 1 5
# Key columns:
# r/s, w/s       IOPS (reads/writes per second)
# rkB/s, wkB/s   Throughput (KB/s)
# await           Average latency (< 10ms good, > 50ms bad)
# r_await         Read latency
# w_await         Write latency
# avgqu-sz        Queue depth (> 1 = requests queueing)
# %util           Busy percentage
#                 HDD: meaningful (> 70% = bottleneck)
#                 SSD: misleading (handles parallel I/O)

# FOR SSDs: focus on await + avgqu-sz, NOT %util
```

### Other Disk Tools

```bash
iotop -o                          # processes doing I/O right now
pidstat -d 1 5                    # per-process disk stats
lsof -p PID                      # files opened by a process
lsof +L1                         # deleted files still consuming space!
# "Disk full but can't find large files" → lsof +L1 shows phantom space

# Emergency disk cleanup:
du -sh /* 2>/dev/null | sort -rh | head -10    # biggest directories
find / -type f -size +1G 2>/dev/null           # giant files
> /var/log/huge.log                             # truncate (don't delete!)
# Truncate preserves the file handle. Delete doesn't free space if process still has it open.
```

### DAY 4 EXERCISES

> **EXERCISE 1:** Run `iostat -xz 1 5`. Which device has highest await? Any queueing?
> **EXERCISE 2:** `dd if=/dev/zero of=/tmp/testfile bs=1M count=1000` — watch iostat during this.
> **EXERCISE 3:** Run iotop. Who's doing the most I/O?
> **EXERCISE 4:** Run pidstat -d 1 5. Which processes read/write most?
> **EXERCISE 5:** What's the difference between %util on HDD vs SSD?
> **EXERCISE 6:** Practice lsof +L1 — when would you use it? ("disk full but files not found")

---

## DAY 5: PRACTICE LAB
**Apr 2, 2025 | 1.5 hrs — COMBINED INVESTIGATION**

> **SCENARIO: Slow Web Server**
>
> Users report 10-second page loads (normal: 200ms).
>
> a) Run 60-second checklist. Record findings.
> b) Build USE table (CPU/Mem/Disk/Network × U/S/E).
> c) Narrow to one resource (which is the bottleneck?).
> d) Deep dive into that resource with specialized tools.
> e) Identify root cause.
> f) Document: symptom → investigation → root cause → fix → prevention.

---

## DAY 6: strace — System Call Tracing
**Apr 3, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: what system calls are, how to trace them
> 00:20-01:10 Hands-on: debug real programs with strace
> 01:10-01:30 Drills: tracing challenges

### strace Basics

```bash
# strace shows EVERY system call a process makes.
# System calls = how programs talk to the kernel (open files, network, etc.)

strace ls -la /etc                   # trace a new command
strace -p PID                        # attach to running process
strace -p PID -e trace=file          # only file operations
strace -p PID -e trace=network       # only network operations
strace -c ls -la /etc                # summary (count per syscall)
strace -T ls -la /etc                # show TIME per call (find slow ones)
strace -t ls -la /etc                # timestamp each call
strace -f -p PID                     # follow child processes too

# COMMON SYSCALLS:
# open/openat   Open a file
# read/write    Read/write data
# close         Close file descriptor
# connect       Connect to network
# socket        Create network socket
# stat/fstat    Get file info
# mmap          Map file into memory
# clone/fork    Create new process
```

### SRE strace Patterns

```bash
# WHY IS THIS PROCESS HANGING?
strace -p PID -e trace=network
# If stuck on connect() or read() → network issue
# If stuck on read() on a file → disk issue or lock

# WHAT FILES IS IT TRYING TO OPEN?
strace -p PID -e trace=open,openat 2>&1 | grep -v "= -1"
# Shows all successfully opened files

# WHAT'S TAKING SO LONG?
strace -c -p PID
# After a few seconds, Ctrl+C. Shows time per syscall.
# If read() dominates → I/O bound
# If futex() dominates → lock contention
# If clock_nanosleep → it's sleeping (normal for a daemon)

# WHAT CONFIG FILES DOES IT READ?
strace -e trace=openat python3 app.py 2>&1 | grep -E "\.conf|\.yaml|\.json"
```

> **⚠️ PRODUCTION WARNING**
>
> strace slows the process 10-100×. Use briefly, then detach.
> For production: use `strace -c` (summary only) or `perf trace` instead.

### DAY 6 EXERCISES

> **EXERCISE 1:** `strace ls -la /etc` — identify open, getdents, write syscalls.
> **EXERCISE 2:** `strace -e trace=openat python3 -c "import os"` — what files does Python open?
> **EXERCISE 3:** `strace -c curl -s https://google.com` — which syscall takes the most time?
> **EXERCISE 4:** Attach strace to a running process. What's it doing?
> **EXERCISE 5:** `strace -T` on a slow command. Find the slowest individual syscall.
> **EXERCISE 6:** A process hangs. How would you use strace to find out what it's waiting for?

---

## DAY 7: perf & Profiling
**Apr 4, 2025 | 1.5 hrs**

### perf — CPU Profiling

```bash
# perf records WHERE a program spends CPU time.

# Record a profile (30 seconds):
perf record -g -p PID -- sleep 30

# Analyze:
perf report                         # interactive report
# Shows function call tree with % CPU time

# Quick stats (IPC, cache misses):
perf stat -p PID -- sleep 10
# IPC < 1 = memory-bound, IPC > 1 = efficient

# Live hottest functions:
perf top -p PID

# System-wide profiling:
perf top                            # live system-wide hot functions
```

### Flame Graphs

```
Flame graphs visualize WHERE time is spent:
- X-axis = function name (alphabetical, NOT time)
- Y-axis = call stack depth
- WIDTH = time spent (wider = more CPU time)

How to read:
- Look at the TOP of the flame (the functions actually running)
- Wide plateaus at the top = optimize THESE functions
- Narrow spikes = not worth optimizing

Generate one:
  perf record -g -p PID -- sleep 30
  perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
  # Open flame.svg in a browser
```

### DAY 7 EXERCISES

> **EXERCISE 1:** `perf stat ls -la /etc` — what's the IPC? Cache miss rate?
> **EXERCISE 2:** `perf record -g -- sleep 10` then `perf report`. What's the hottest function?
> **EXERCISE 3:** Run `perf top` — what functions consume the most CPU system-wide?
> **EXERCISE 4:** Explain flame graphs: what do width and height mean?

---

## DAY 8: KERNEL TUNING
**Apr 5, 2025 | 1.5 hrs**

### sysctl — Runtime Kernel Parameters

```bash
# View all:
sysctl -a | wc -l                    # hundreds of parameters!

# View specific:
sysctl vm.swappiness
sysctl net.core.somaxconn

# Set temporarily:
sudo sysctl vm.swappiness=10
sudo sysctl net.core.somaxconn=65535

# Set permanently:
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.d/99-custom.conf
sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

### Production Tuning Cheat Sheet

```bash
# === MEMORY ===
vm.swappiness = 10                     # prefer RAM over swap (databases!)
vm.dirty_ratio = 10                    # write-back threshold (% RAM)
vm.dirty_background_ratio = 5          # background write-back starts

# === NETWORK (high-traffic servers) ===
net.core.somaxconn = 65535             # max listen backlog
net.ipv4.tcp_max_syn_backlog = 65535   # SYN queue size
net.ipv4.tcp_fin_timeout = 30          # faster FIN cleanup (default 60)
net.ipv4.tcp_tw_reuse = 1             # reuse TIME_WAIT sockets
net.ipv4.ip_local_port_range = 1024 65535  # more ephemeral ports

# === FILE DESCRIPTORS ===
fs.file-max = 2097152                  # system-wide FD limit
# Per-process: /etc/security/limits.conf:
#   * soft nofile 65535
#   * hard nofile 65535
# Or in systemd: LimitNOFILE=65535

# === APPLY ===
sudo sysctl -p /etc/sysctl.d/99-custom.conf
```

### Common Errors and Their Sysctl Fix

```
Error                              Fix
──────────────────────────────     ──────────────────────────────────
"Too many open files"              fs.file-max + ulimit -n
"Connection refused" under load    net.core.somaxconn
"Cannot assign requested address"  ip_local_port_range + tcp_tw_reuse
OOM kills                          vm.overcommit_memory + process limits
High swap on database              vm.swappiness=10
```

### DAY 8 EXERCISES

> **EXERCISE 1:** Check current values for: swappiness, somaxconn, file-max, tcp_fin_timeout.
> **EXERCISE 2:** Create /etc/sysctl.d/99-sre.conf with the production tuning values above.
> **EXERCISE 3:** A server gets "too many open files". Debug it: check fs.file-max, ulimit -n, lsof count.
> **EXERCISE 4:** Explain: why set swappiness=10 for databases?
> **EXERCISE 5:** What does tcp_tw_reuse do? When is it needed?

---

## DAY 9: BENCHMARKING
**Apr 7, 2025 | 1.5 hrs**

### Benchmarking Methodology

```
Golden rule: change ONE thing at a time, measure, compare.

1. Establish BASELINE (before any changes)
2. Change ONE variable
3. Re-measure
4. Compare with baseline
5. Repeat for each variable

Mistakes:
- Changing multiple things at once (can't tell what helped)
- Not warming up (first run often slower — cache cold)
- Not running enough iterations (need statistical confidence)
```

### Benchmark Tools

```bash
# CPU:
stress-ng --cpu 4 --timeout 60s                    # stress 4 cores
stress-ng --cpu 4 --cpu-method matrixprod --metrics # with metrics
sysbench cpu --threads=4 --time=30 run             # standardized benchmark

# MEMORY:
stress-ng --vm 2 --vm-bytes 2G --timeout 60s       # memory stress
sysbench memory --threads=4 --time=30 run

# DISK:
fio --name=randread --rw=randread --bs=4k --size=1G --numjobs=4 --runtime=30
fio --name=seqwrite --rw=write --bs=1M --size=1G --numjobs=1 --runtime=30
# Key output: IOPS, bandwidth, latency (avg, p95, p99)

# NETWORK:
iperf3 -s                          # server side
iperf3 -c server_ip -t 30         # client side (30 seconds)
# Shows: bandwidth, jitter, packet loss

# HTTP:
wrk -t4 -c100 -d30s http://localhost:8080/
# Shows: requests/sec, latency (avg, p50, p99), transfer rate
ab -n 10000 -c 100 http://localhost:8080/
# Apache Bench: simpler alternative
```

### DAY 9 EXERCISES

> **EXERCISE 1:** Baseline your system: stress-ng --cpu, --vm, fio for disk. Record results.
> **EXERCISE 2:** Change swappiness from 60→10. Re-run memory benchmark. Any difference?
> **EXERCISE 3:** Run fio random read. What IOPS and latency does your disk achieve?
> **EXERCISE 4:** wrk or ab against a web server. What's the max requests/sec?
> **EXERCISE 5:** Run the same benchmark 5 times. How much variation? (Need enough runs for confidence.)

---

## DAY 10: CAPSTONE — FULL PERFORMANCE INVESTIGATION
**Apr 8-9, 2025 | 1.5 hrs each**

> **SCENARIO: Database Performance Degradation**
>
> PostgreSQL query throughput dropped from 5000 to 500 queries/sec.
> Application logs show "query timeout" errors.
>
> **PHASE 1: 60-Second Checklist (10 min)**
> Run all 10 commands. Record findings.
>
> **PHASE 2: USE Table (10 min)**
> Build the full USE table for CPU/Memory/Disk/Network.
> Which resource is the bottleneck?
>
> **PHASE 3: Deep Dive (20 min)**
> Based on Phase 2, deep dive into the bottleneck:
> - CPU: mpstat, pidstat, perf
> - Memory: /proc/meminfo, vmstat si/so, OOM check
> - Disk: iostat, iotop, lsof
> - Network: ss, tcpdump, mtr
>
> **PHASE 4: strace/perf (15 min)**
> Attach to the database process:
> - strace -c to find where time is spent
> - perf record for CPU profile
>
> **PHASE 5: Kernel Check (10 min)**
> Check sysctl values: swappiness, somaxconn, file-max.
> Any obvious misconfigurations?
>
> **PHASE 6: Root Cause & Document (15 min)**
> Write incident report: symptom → investigation → root cause → fix → prevention.

---

## WEEKS 7-8 SUMMARY

> **FILE 04 COMPLETE — LINUX PERFORMANCE**
>
> ✓ USE Method: Utilization, Saturation, Errors for every resource
> ✓ 60-second checklist: 10 commands for first response
> ✓ CPU: user/system/iowait/steal, load avg, mpstat, pidstat
> ✓ Memory: free vs available, page cache, swap, OOM killer
> ✓ Disk: IOPS/throughput/latency, iostat, iotop, lsof +L1
> ✓ strace: system call tracing, debugging hangs
> ✓ perf: CPU profiling, flame graphs
> ✓ Kernel tuning: sysctl, production settings, common errors
> ✓ Benchmarking: stress-ng, fio, wrk, methodology
> ✓ 50+ exercises including full investigation capstone
>
> **INTERVIEW PREP:**
> - "Server is slow. Walk me through debugging." → 60-second checklist,
>   then USE method: check CPU, memory, disk, network systematically.
> - "What is the USE method?" → For each resource check Utilization,
>   Saturation, Errors. If all OK → resource is fine, move on.
> - "What does high iowait mean?" → Disk I/O bottleneck, NOT CPU.
>   Check iostat for await and queue depth.
> - "App killed with no logs." → OOM killer. Check dmesg.
> - "Disk full but can't find files." → lsof +L1 for deleted-but-open files,
>   or df -i for inode exhaustion.
>
> Next: File 05 — Linux Security & Hardening (Weeks 9-10, Apr 12-23)
