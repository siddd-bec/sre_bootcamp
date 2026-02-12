# LINUX & SYSTEMS — FILE 04 of 48
## LINUX PERFORMANCE
### CPU, Memory, Disk I/O, Process Tracing, Kernel Tuning, Benchmarking
**Weeks 7-8 | Mar 29 - Apr 9, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY PERFORMANCE SEPARATES GOOD SREs FROM GREAT ONES**
>
> Any SRE can restart a service. Great SREs find WHY it needed restarting.
> Apple interview: "A server is slow. Walk me through your debugging process."
> THIS FILE is your answer.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | The USE Method | Utilization, Saturation, Errors framework |
| 2 | CPU Deep Dive | user/system/iowait/steal, load avg, mpstat |
| 3 | Memory Deep Dive | RSS/VSZ, page cache, swap, OOM killer |
| 4 | Disk I/O | IOPS, throughput, latency, iostat, iotop |
| 5 | Practice Lab I | Combined performance investigation |
| 6 | strace and ltrace | System call tracing, debugging hangs |
| 7 | perf and Profiling | CPU profiling, flame graphs, hotspots |
| 8 | Kernel Tuning | sysctl, /proc/sys, production settings |
| 9 | Benchmarking | stress-ng, fio, sysbench, methodology |
| 10 | Capstone | Full performance investigation scenario |

---

## DAY 1: THE USE METHOD
**Mar 29, 2025 | 1.5 hrs**

> **THE USE METHOD — BY BRENDAN GREGG**
>
> For every resource (CPU, memory, disk, network) check:
> - **U = Utilization:** how busy is it? (percent of capacity)
> - **S = Saturation:** is work queuing? (beyond capacity)
> - **E = Errors:** any error events?
>
> If U less than 70% AND S = 0 AND E = 0 then resource is fine.
> If any are bad then you found the bottleneck.

### USE Method Applied

```bash
# CPU:
#   Utilization: top -> %us + %sy + %si (under 70% OK)
#   Saturation:  vmstat -> column 'r' (run queue under 2x cores)
#   Errors:      perf stat -> cache misses, MCE errors

# MEMORY:
#   Utilization: free -h -> used/total
#   Saturation:  vmstat -> si/so (swap in/out, should be 0)
#   Errors:      dmesg | grep -i 'oom|out of memory'

# DISK:
#   Utilization: iostat -x -> %util (under 70% for HDDs)
#   Saturation:  iostat -x -> avgqu-sz (queue over 1 = queueing)
#   Errors:      dmesg | grep -i 'error|fault|failed'

# NETWORK:
#   Utilization: sar -n DEV -> rxkB/s, txkB/s vs link speed
#   Saturation:  netstat -s | grep retrans
#   Errors:      ip -s link show -> errors, drops
```

> **THE FIRST 60 SECONDS — BRENDAN GREGG CHECKLIST**
>
> 1. `uptime` — load averages
> 2. `dmesg -T | tail` — kernel errors (OOM? disk?)
> 3. `vmstat 1 5` — CPU, memory, I/O overview
> 4. `mpstat -P ALL 1` — per-CPU breakdown
> 5. `pidstat 1 5` — per-process resource usage
> 6. `iostat -xz 1 5` — disk I/O per device
> 7. `free -h` — memory (available is key!)
> 8. `sar -n DEV 1 5` — network throughput
> 9. `sar -n TCP 1 5` — TCP connection rates
> 10. `top` — overall picture
>
> Memorize this. It is your muscle memory during incidents.

**Exercises:** Run the checklist, identify USE metrics per output, create USE table, explain vmstat columns, interpret load average, write USE script.

---

## DAY 2: CPU DEEP DIVE
**Mar 30, 2025 | 1.5 hrs**

```bash
# CPU time from top or mpstat:
# %us (user):    application code
# %sy (system):  kernel (syscalls, interrupts)
# %wa (iowait):  waiting for I/O — NOT a CPU problem!
# %st (steal):   stolen by hypervisor (VM only!)

# High %us -> CPU-bound  |  High %sy -> syscall heavy
# High %wa -> disk issue  |  High %st -> VM throttled

mpstat -P ALL 1         # per-CPU utilization
pidstat -u 1 5          # per-process CPU
top -H -p PID           # per-thread CPU
pidstat -w 1            # context switches per process
```

> **CPU STEAL — THE CLOUD GOTCHA**
> %st means hypervisor giving CPU to other VMs. App slow but CPU looks low. Fix: bigger instance.

**Exercises:** mpstat, stress-ng CPU load, dd iowait, pidstat, per-thread top, context switch analysis, load avg interpretation.

---

## DAY 3: MEMORY DEEP DIVE
**Mar 31, 2025 | 1.5 hrs**

```bash
# free -h: 'available' matters, NOT 'free'!
# Linux uses free RAM for cache. Cache is instantly reclaimable.
# RSS (Resident Set Size) = physical RAM used NOW <- this matters
# VSZ (Virtual Size) = virtual space (includes mapped files)

ps aux --sort=-%mem | head -10
cat /proc/meminfo   # MemAvailable, Cached, SwapFree, Dirty, Slab
```

> **OOM KILLER**
> RAM + swap exhausted -> kernel kills a process.
> Check: `dmesg | grep -i 'oom|killed process'`
> Protect: `echo -1000 > /proc/PID/oom_score_adj`
> Swap: swappiness=10 for databases, 60 default.
> Interview: "App killed no logs." -> "OOM. Check dmesg."

**Exercises:** free -h analysis, /proc/meminfo, stress --vm, watch cache, swappiness, dmesg OOM, oom_score.

---

## DAY 4: DISK I/O
**Apr 1, 2025 | 1.5 hrs**

```bash
# iostat -xz 1 (THE SRE disk tool):
# await:    avg latency (under 10ms good, over 50ms bad)
# %util:    busy (HDD meaningful, SSD misleading)
# avgqu-sz: queue depth (over 1 = queueing)

iotop -o              # processes doing I/O
pidstat -d 1 5        # per-process disk stats
lsof -p PID           # open files
```

> **SSD vs HDD:** HDD 75-200 IOPS, use %util. SSD 10K-500K IOPS, use await + queue.

**Exercises:** iostat, dd load, iotop, pidstat -d, SSD vs HDD metrics, lsof on database.

---

## DAY 5: PRACTICE LAB
**Apr 2, 2025 — COMBINED INVESTIGATION**

Slow web server, 10s page loads: 60-second checklist, USE table, narrow to resource, root cause, document.

---

## DAY 6: strace
**Apr 3, 2025 | 1.5 hrs**

```bash
strace -p PID                  # trace running process
strace -p PID -e trace=file    # only file ops
strace -c ls -la /             # summary per syscall
strace -T ls -la /             # time per call
```

> **PRODUCTION:** strace slows 10-100x. Use `perf trace` or `strace -c` instead.

**Exercises:** strace ls, file opens, curl network, Python import, slowest syscall, debug hangs.

---

## DAY 7: perf and Profiling
**Apr 4, 2025 | 1.5 hrs**

```bash
perf record -g -p PID -- sleep 30  # record profile
perf report                         # analyze
perf stat -p PID -- sleep 10       # IPC, cache misses
perf top -p PID                    # live hot functions
# Flame graphs: width = time, wide tops = optimize here
```

---

## DAY 8: KERNEL TUNING
**Apr 5, 2025 | 1.5 hrs**

```bash
# MEMORY: vm.swappiness=10, vm.dirty_ratio=10
# NETWORK: somaxconn=65535, tcp_fin_timeout=30, tcp_tw_reuse=1
# FDs: fs.file-max=2097152, LimitNOFILE=65535
# Persistent: /etc/sysctl.d/99-custom.conf then sysctl -p
```

> 1. "Too many open files" -> fs.file-max + ulimit
> 2. "Connection refused" -> somaxconn
> 3. "Out of ports" -> ip_local_port_range + tcp_tw_reuse
> 4. "OOM kills" -> overcommit_memory + limits
> 5. "High swap" -> swappiness=10

---

## DAY 9: BENCHMARKING
**Apr 7, 2025 | 1.5 hrs**

```bash
# Methodology: baseline -> ONE change -> remeasure -> compare
stress-ng --cpu 4 --timeout 60s         # CPU
fio --rw=randread --bs=4k --size=1G     # disk IOPS
iperf3 -c server -t 30                  # network
wrk -t4 -c100 -d30s http://localhost/   # HTTP
```

---

## DAY 10: CAPSTONE
**Apr 8, 2025 — DATABASE DEGRADATION**

PostgreSQL 5000 -> 500 qps. Run checklist, USE table, deep dive, strace/perf, kernel check, root cause, document.

---

> **FILE 04 COMPLETE — LINUX PERFORMANCE**
> Next: File 05 — Linux Security and Hardening (Weeks 9-10, Apr 12-23)
