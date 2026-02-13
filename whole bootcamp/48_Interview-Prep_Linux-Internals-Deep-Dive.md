# INTERVIEW PREP — FILE 48 of 48
## LINUX INTERNALS & SYSTEMS DEEP DIVE
### Zombie Processes, Inodes, Signals, Kernel Boot, System Calls, TCP vs UDP, Swap, Garbage Collection
**Reference Document | Use alongside Files 01-05 | Study: 3-5 days at your own pace**

---

> **WHAT THIS FILE IS**
>
> These are the questions that separate "I can use Linux" from
> "I understand how Linux WORKS under the hood."
>
> Apple, Google, Meta SRE interviews love these questions because
> they test whether you can troubleshoot problems you've never
> seen before — which requires understanding the internals.
>
> Each question gets a full explanation: what it is, why it matters,
> how it works at the kernel level, and how to answer in an interview.

| # | Topic | Interview Question |
|---|-------|-------------------|
| 1 | Zombie Processes | What is a zombie process? How do you find and fix them? |
| 2 | Hard vs Soft Links | What's the difference? When would you use each? |
| 3 | Inodes | What is an inode? What happens when you run out? |
| 4 | Signals | What's a signal? How does the kernel handle it? |
| 5 | Default Kill Signal | What signal does `kill` send by default? |
| 6 | Process Termination | How do you terminate a running process? |
| 7 | TCP vs UDP | What's the difference? When would you use each? |
| 8 | Swap | What is swap space? When is it used? |
| 9 | Kernel Boot Process | How does a Linux kernel load? |
| 10 | `ls -l` at Kernel Level | What happens when you type `ls -l`? |
| 11 | fork() Return Value | Which system call returns non-zero on success? |
| 12 | Garbage Collection | How does a garbage collector work? |

---

## 1. ZOMBIE PROCESSES

### Interview Question: "What is a zombie process? How do you find and fix them?"

### What Is a Zombie Process?

A zombie is a process that has **finished executing** but still has an entry in the **process table**. It's dead, but its "death certificate" hasn't been collected yet.

```
PROCESS LIFECYCLE:
                                                    Parent calls
  fork() → Child runs → Child exits → ZOMBIE → wait()/waitpid() → GONE
                              ↑                        ↑
                         Child is done            Parent reads
                         but parent hasn't        the exit status
                         read exit status         and cleans up
```

When a process exits, the kernel keeps a small entry in the process table containing:
- The **exit status** (did it succeed or fail?)
- The **PID**
- Some accounting info (CPU time used, etc.)

This entry exists so the PARENT process can call `wait()` or `waitpid()` to retrieve the child's exit status. Until the parent does this, the child is a **zombie** (state `Z`).

### Why Zombies Exist

```python
# SIMPLIFIED MENTAL MODEL:

# Step 1: Parent creates child
parent_pid = 1000
child_pid = fork()    # kernel creates child process (PID 1001)

# Step 2: Child does its work and exits
# Child calls exit(0) → kernel marks it as zombie (Z)
# Kernel sends SIGCHLD signal to parent

# Step 3: Parent SHOULD call wait()
exit_status = waitpid(child_pid)   # reads exit code, kernel removes zombie
# But if parent IGNORES this... zombie stays forever
```

### Why Zombies Are a Problem

```
A single zombie uses almost no memory or CPU.
But EACH zombie holds a PID and a process table slot.

The system has a LIMITED number of PIDs (default: 32768).
$ cat /proc/sys/kernel/pid_max
32768

If zombies pile up, the system RUNS OUT OF PIDs.
New processes CAN'T start. Fork fails. Services crash.

This is a real production incident scenario.
```

### How to Find Zombies

```bash
# Find zombie processes:
ps aux | grep Z
# Or more precisely:
ps -eo pid,ppid,stat,cmd | grep -w Z

# Output example:
#   PID  PPID STAT CMD
#  5432  5430 Z    [my-worker] <defunct>

# Count zombies:
ps -eo stat | grep -c Z

# KEY: Note the PPID (parent PID) — that's the REAL problem.
# The zombie itself is harmless. The parent that isn't calling wait() is the bug.

# Find the parent:
ps -p 5430 -o pid,cmd
# Or:
cat /proc/5430/cmdline
```

### How to Fix Zombies

```bash
# Option 1: Fix the parent (BEST solution)
# The parent process has a bug — it's not calling wait().
# Fix the code: add a SIGCHLD handler or call waitpid().

# Option 2: Send SIGCHLD to the parent
kill -SIGCHLD 5430
# This reminds the parent to reap its children.
# Only works if parent has a SIGCHLD handler.

# Option 3: Kill the PARENT (not the zombie!)
kill 5430
# You CAN'T kill a zombie — it's already dead!
# kill -9 5432  ← THIS DOES NOTHING. Zombie is already dead.
# Killing the parent causes zombies to be adopted by init (PID 1),
# which automatically calls wait() and cleans them up.

# Option 4: Wait for init to reap
# If the parent dies for any reason, init (PID 1) adopts orphans
# and reaps zombies. Systemd does this automatically.
```

### The Code Fix (Python Example)

```python
import os
import signal

# BAD — creates zombies:
def bad_parent():
    for i in range(100):
        pid = os.fork()
        if pid == 0:
            # Child process
            os._exit(0)    # child exits → becomes zombie!
        # Parent never calls wait() → zombies pile up

# GOOD — properly reaps children:
def good_parent():
    # Method 1: Explicit wait
    for i in range(100):
        pid = os.fork()
        if pid == 0:
            os._exit(0)
        os.waitpid(pid, 0)    # reap immediately

    # Method 2: SIGCHLD handler (for async children)
    signal.signal(signal.SIGCHLD, lambda s, f: os.waitpid(-1, os.WNOHANG))

    # Method 3: Ignore SIGCHLD (kernel auto-reaps)
    signal.signal(signal.SIGCHLD, signal.SIG_IGN)
```

### Interview Answer (30 seconds)

> "A zombie is a process that's finished executing but still has a process table entry because its parent hasn't called wait() to read its exit status. You can't kill a zombie — it's already dead. The fix is to either fix the parent's code to properly reap children, send SIGCHLD to the parent, or kill the parent so init adopts and reaps the zombies. In production, a few zombies are harmless, but if they pile up they can exhaust the PID space and prevent new processes from starting."

---

## 2. HARD LINKS vs SOFT LINKS

### Interview Question: "What's the difference between a hard link and a soft link?"

### First: Understand Inodes (This is #6 too)

```
Every file on a Linux filesystem is really TWO things:

1. The INODE — a data structure that stores:
   - File size, permissions, owner, timestamps
   - Pointers to the actual data blocks on disk
   - Does NOT store the filename!

2. The DIRECTORY ENTRY — maps a name to an inode number
   /home/sid/myfile.txt → inode #12345

The filename is just a LABEL pointing to an inode.
Multiple labels can point to the same inode. That's a hard link.
```

### Hard Link

```
A hard link is another directory entry pointing to the SAME inode.

  /home/sid/file1.txt  ──→  inode #12345  ──→  [data blocks on disk]
  /home/sid/file2.txt  ──→  inode #12345  ──→  [same data blocks!]

Both names point to the SAME data. There's no "original" and "copy."
They are EQUAL references to the same underlying file.
```

```bash
# Create hard link:
ln /home/sid/file1.txt /home/sid/file2.txt

# Verify — same inode number:
ls -li /home/sid/file1.txt /home/sid/file2.txt
# 12345 -rw-r--r-- 2 sid sid 1024 May 20 file1.txt
# 12345 -rw-r--r-- 2 sid sid 1024 May 20 file2.txt
#  ↑                ↑
#  same inode       link count = 2

# If you delete file1.txt, the data STILL EXISTS via file2.txt.
# Data is only deleted when link count reaches 0.

# The inode tracks the link count:
stat file1.txt | grep Links    # Links: 2
```

### Soft Link (Symbolic Link / Symlink)

```
A soft link is a SPECIAL FILE that contains a PATH to another file.
It's like a shortcut or pointer.

  /home/sid/link.txt  ──→  "/home/sid/original.txt"  ──→  inode #12345  ──→  [data]
       (symlink)            (just stores this path)

The symlink has its OWN inode. It just stores a path string.
```

```bash
# Create soft link:
ln -s /home/sid/original.txt /home/sid/link.txt

# Verify — different inodes:
ls -li
# 12345 -rw-r--r-- 1 sid sid 1024 May 20 original.txt
# 67890 lrwxrwxrwx 1 sid sid   27 May 20 link.txt -> /home/sid/original.txt
#  ↑                                         ↑
#  different inode!                           shows target path

# If you delete original.txt, link.txt becomes a BROKEN LINK:
rm original.txt
cat link.txt    # ERROR: No such file or directory (dangling symlink)
```

### Comparison Table

```
Feature              | Hard Link              | Soft Link (Symlink)
---------------------|------------------------|------------------------
Points to            | Same inode (data)      | A pathname (string)
Own inode?           | No (shares inode)      | Yes (has its own inode)
Cross filesystem?    | NO (same FS only)      | YES (can cross filesystems)
Link to directory?   | NO (usually forbidden) | YES
If target deleted?   | Data survives          | BROKEN link (dangling)
File size            | Same as original       | Size of the path string
Permissions          | Same as original       | Always lrwxrwxrwx
Link count affected? | YES (increments)       | NO
```

### When to Use Each

```
USE HARD LINKS:
- Backup systems (save space, both names are equal)
- When you need the file to survive if "original" is deleted
- Same filesystem only

USE SOFT LINKS (more common):
- Linking across filesystems or partitions
- Linking to directories (/var/log/app → /mnt/logs/app)
- Version management (/usr/bin/python → /usr/bin/python3.10)
- Config management (/etc/nginx/sites-enabled/mysite → ../sites-available/mysite)

SRE uses symlinks constantly:
  /opt/app/current → /opt/app/release-v2.3.1
  (deploy new version by changing the symlink — instant rollback!)
```

### Interview Answer (30 seconds)

> "A hard link is another directory entry pointing to the same inode — both names are equal references to the same data, and the data persists until the last hard link is deleted. A soft link is a separate file that stores a path to another file — it can cross filesystems and link to directories, but breaks if the target is deleted. In SRE, we use symlinks heavily for deployments — point /opt/app/current to the release directory, and rollback by changing the symlink."

---

## 3. INODES IN DETAIL

### Interview Question: "What is the function of inodes in a Linux filesystem?"

### Inode Structure

```
An inode is a data structure on disk that stores ALL metadata about a file
EXCEPT its name. Every file and directory has exactly one inode.

INODE CONTENTS:
┌─────────────────────────────────────┐
│ Inode #12345                        │
├─────────────────────────────────────┤
│ File type (regular, directory, etc.)│
│ Permissions (rwxr-xr-x)            │
│ Owner UID / Group GID              │
│ File size (bytes)                   │
│ Timestamps:                         │
│   - atime (last accessed)          │
│   - mtime (last modified)          │
│   - ctime (last inode changed)     │
│ Link count (hard links)            │
│ Pointers to data blocks:           │
│   - 12 direct pointers             │
│   - 1 single indirect pointer      │
│   - 1 double indirect pointer      │
│   - 1 triple indirect pointer      │
└─────────────────────────────────────┘

NOT in the inode: the filename!
Filenames live in DIRECTORY entries.
A directory is just a table: name → inode number.
```

### How Files Are Found

```
When you access /home/sid/report.txt:

1. Kernel reads inode #2 (root directory /)
2. Looks up "home" in /'s directory entries → inode #100
3. Reads inode #100 (/home directory)
4. Looks up "sid" in /home's entries → inode #200
5. Reads inode #200 (/home/sid directory)
6. Looks up "report.txt" → inode #12345
7. Reads inode #12345 → gets data block pointers
8. Reads the actual data blocks from disk

This is why deeply nested paths are slightly slower —
more directory lookups needed.
```

### Running Out of Inodes (Classic SRE Interview!)

```bash
# A filesystem has a FIXED number of inodes, set at creation time.
# You can run out of inodes EVEN WITH FREE DISK SPACE.

# Check inode usage:
df -i
# Filesystem     Inodes   IUsed  IFree IUse% Mounted on
# /dev/sda1     6553600  6553598     2  100% /

# Disk still has space, but NO FREE INODES!
# Can't create any new files.

# Common cause: millions of tiny files
# Example: a log rotation bug creates millions of 0-byte files
# Example: a mail queue fills with millions of small messages

# Fix:
find /tmp -type f -name "*.tmp" -mtime +7 -delete    # delete old files
# Or recreate filesystem with more inodes:
# mkfs.ext4 -N 20000000 /dev/sda1    # specify inode count at creation
```

### Interview Answer (20 seconds)

> "An inode stores all file metadata — permissions, ownership, timestamps, size, and pointers to data blocks — everything except the filename. Filenames are stored in directory entries that map to inode numbers. A key SRE issue is inode exhaustion: if millions of tiny files consume all inodes, you can't create new files even with free disk space. You diagnose this with `df -i`."

---

## 4. SIGNALS & KERNEL SIGNAL HANDLING

### Interview Question: "What's a signal, and how is it handled by the kernel?"

### What Is a Signal?

```
A signal is a SOFTWARE INTERRUPT sent to a process.
It's the kernel's way of saying "hey, something happened!"

Signals can come from:
- The kernel (SIGSEGV when you access bad memory)
- Another process (kill command)
- The user (Ctrl+C sends SIGINT)
- The process itself (abort())
- Hardware events (SIGFPE for divide by zero)
```

### Common Signals Table

```
Signal    Number  Default Action    Meaning
────────  ──────  ────────────────  ──────────────────────────────────
SIGHUP      1     Terminate         Terminal hangup (also: reload config)
SIGINT      2     Terminate         Ctrl+C — interrupt from keyboard
SIGQUIT     3     Core dump         Ctrl+\ — quit with core dump
SIGKILL     9     Terminate         CANNOT be caught or ignored — forced kill
SIGTERM    15     Terminate         Polite "please exit" — THE DEFAULT for kill
SIGCHLD    17     Ignore            Child process stopped or terminated
SIGSTOP    19     Stop              CANNOT be caught — pause process
SIGCONT    18     Continue          Resume a stopped process
SIGSEGV    11     Core dump         Segmentation fault (bad memory access)
SIGPIPE    13     Terminate         Write to pipe with no reader
SIGUSR1    10     Terminate         User-defined signal 1
SIGUSR2    12     Terminate         User-defined signal 2
SIGALRM    14     Terminate         Alarm clock timer expired
```

### How the Kernel Handles Signals — Step by Step

```
1. SIGNAL IS SENT
   - kill(pid, SIGTERM) system call is invoked
   - Kernel marks signal as "pending" in the target process's
     signal bitmap (each process has a bitmask of pending signals)

2. SIGNAL IS DELIVERED
   - Before the process returns to user mode (e.g., after a
     system call or timer interrupt), kernel checks pending signals
   - If a signal is pending and not blocked, kernel delivers it

3. SIGNAL IS HANDLED (one of three outcomes):
   a) DEFAULT action — kernel does what the signal table says
      (terminate, core dump, stop, ignore, or continue)
   
   b) CUSTOM HANDLER — process registered a handler function
      - Kernel saves the process's current context (registers, etc.)
      - Switches to the signal handler function
      - Handler runs in USER space
      - When handler returns, kernel restores original context
   
   c) IGNORED — process called signal(SIGTERM, SIG_IGN)
      - Kernel discards the signal
      - Exception: SIGKILL and SIGSTOP can NEVER be ignored

4. SIGNAL MASK
   - Processes can BLOCK signals temporarily (sigprocmask)
   - Blocked signals stay pending until unblocked
   - This prevents race conditions in critical sections
   - SIGKILL and SIGSTOP cannot be blocked
```

### Why SIGKILL (-9) Is Special

```
SIGKILL is the kernel's "emergency stop button."

Normal signals:          SIGKILL:
  Process → Handler →    Kernel terminates immediately.
  cleanup → exit         No handler. No cleanup. No saving state.
                         No chance to close files, flush buffers,
                         release locks, or write "shutting down" to log.

This is why kill -9 is a LAST RESORT:
  1. First try: kill PID           (SIGTERM — graceful shutdown)
  2. Wait 5-10 seconds
  3. If still alive: kill -9 PID   (SIGKILL — force kill)

Using kill -9 first can cause:
  - Corrupted files (not flushed)
  - Zombie children (parent killed before reaping)
  - Held locks not released (deadlocking other processes)
  - Lost data in buffers
```

### SRE Signal Use Cases

```bash
# Reload config without restart (SIGHUP):
kill -HUP $(cat /var/run/nginx.pid)
# nginx re-reads config, no downtime!

# Graceful shutdown (SIGTERM):
kill $(cat /var/run/myapp.pid)
# App catches SIGTERM, finishes current requests, closes connections

# Core dump for debugging (SIGQUIT):
kill -QUIT $(pidof java)
# Java dumps thread stacks — invaluable for debugging hangs

# Rotate logs (SIGUSR1 — used by nginx):
kill -USR1 $(cat /var/run/nginx.pid)
# nginx reopens log files — used during log rotation
```

### Interview Answer (30 seconds)

> "A signal is a software interrupt delivered to a process by the kernel. When sent, the kernel marks it as pending in the process's signal bitmap. On the next return to user mode, the kernel delivers it. The process can handle it with a custom handler, let the default action occur, or ignore it — except for SIGKILL and SIGSTOP, which cannot be caught or ignored. In SRE, we use SIGTERM for graceful shutdown, SIGHUP for config reload, and SIGKILL only as a last resort since it prevents cleanup."

---

## 5. DEFAULT KILL SIGNAL

### Interview Question: "What is the default signal when you run `kill PID`?"

### Answer

```bash
# The default signal is SIGTERM (signal 15).

kill 1234            # sends SIGTERM (15) — this is the default
kill -15 1234        # same thing, explicit
kill -TERM 1234      # same thing, by name
kill -SIGTERM 1234   # same thing

# SIGTERM = "please terminate gracefully"
# The process CAN catch this and do cleanup before exiting.

# kill -9 is SIGKILL — "terminate NOW, no questions asked"
# Only use when SIGTERM doesn't work.

# Common SRE workflow:
kill $PID                    # Step 1: ask nicely (SIGTERM)
sleep 10                     # Step 2: give it time
kill -0 $PID 2>/dev/null     # Step 3: check if still alive
if [ $? -eq 0 ]; then        # still running...
    kill -9 $PID             # Step 4: force kill
fi
```

---

## 6. TERMINATING A RUNNING PROCESS

### Interview Question: "How do you terminate a running process?"

```bash
# Method 1: By PID
kill 1234                    # SIGTERM (graceful)
kill -9 1234                 # SIGKILL (force)

# Method 2: By name
pkill nginx                  # SIGTERM to all nginx processes
pkill -9 nginx               # SIGKILL to all
killall nginx                # same idea (slightly different matching)

# Method 3: By pattern
pkill -f "python3 my_script.py"    # match full command line

# Method 4: Interactive
top                          # press 'k', enter PID, enter signal (15 or 9)
htop                         # select process, press F9, choose signal

# Method 5: From /proc
# Check if process exists:
kill -0 1234                 # sends signal 0 (null) — just checks existence
# Returns 0 if process exists, non-zero if not

# Method 6: Graceful with timeout (production SRE pattern)
timeout 30 bash -c 'kill $PID; while kill -0 $PID 2>/dev/null; do sleep 1; done' \
  || kill -9 $PID

# Method 7: Process group (kill parent + all children)
kill -TERM -$PGID            # negative PID = process group
# Useful for killing a shell script AND all its spawned children
```

### Interview Answer (15 seconds)

> "Send SIGTERM first with `kill PID` for graceful shutdown — the process can catch it and clean up. If it doesn't respond within a few seconds, escalate to `kill -9 PID` (SIGKILL) which can't be caught. For multiple processes, use `pkill` by name. In production, always try graceful first to avoid data corruption."

---

## 7. TCP vs UDP

### Interview Question: "What's the difference between TCP and UDP?"

### TCP (Transmission Control Protocol)

```
TCP = RELIABLE, ORDERED, CONNECTION-ORIENTED

How it works:

1. THREE-WAY HANDSHAKE (connection setup):
   Client → SYN        → Server       "I want to connect"
   Client ← SYN+ACK    ← Server       "OK, I acknowledge"
   Client → ACK         → Server       "Connection established"

2. DATA TRANSFER:
   - Every segment gets a sequence number
   - Receiver sends ACK for each received segment
   - If sender doesn't get ACK → retransmits
   - Receiver reorders segments if they arrive out of order
   - Flow control: receiver tells sender how much it can handle
   - Congestion control: sender slows down if network is busy

3. FOUR-WAY TEARDOWN (connection close):
   Client → FIN         → Server       "I'm done sending"
   Client ← ACK         ← Server       "Got it"
   Client ← FIN         ← Server       "I'm done too"
   Client → ACK         → Server       "OK, bye"

GUARANTEES:
  ✓ Data arrives
  ✓ Data arrives in order
  ✓ Data is not duplicated
  ✓ Data is not corrupted (checksum)
  ✗ Slower (overhead from handshakes, ACKs, retransmission)
```

### UDP (User Datagram Protocol)

```
UDP = FAST, SIMPLE, CONNECTIONLESS

How it works:
1. No handshake — just send the data (datagram)
2. No acknowledgment — sender doesn't know if receiver got it
3. No ordering — datagrams may arrive out of order
4. No retransmission — lost packets are gone

GUARANTEES:
  ✓ Fast (minimal overhead)
  ✓ Simple
  ✓ Checksum (optional integrity check)
  ✗ Data might not arrive
  ✗ Data might arrive out of order
  ✗ Data might be duplicated
```

### Comparison

```
Feature          | TCP                    | UDP
─────────────────|──────────────────────  |─────────────────────
Connection       | Connection-oriented    | Connectionless
Reliability      | Guaranteed delivery    | Best-effort (may lose)
Ordering         | In-order delivery      | No ordering
Speed            | Slower (overhead)      | Faster (no overhead)
Header size      | 20-60 bytes            | 8 bytes
Flow control     | Yes (window-based)     | No
Congestion ctrl  | Yes                    | No
Use case         | Web, email, file xfer  | DNS, video, gaming
```

### When to Use Each (SRE Perspective)

```
TCP (need reliability):
  - HTTP/HTTPS (web traffic)
  - SSH (remote access)
  - Database connections (PostgreSQL, MySQL)
  - File transfers (SCP, SFTP)
  - SMTP (email)
  - Any time losing data is unacceptable

UDP (need speed, can tolerate loss):
  - DNS lookups (small, fast queries)
  - Monitoring metrics (Prometheus remote write, StatsD)
  - Video/audio streaming (a dropped frame is fine)
  - Online gaming (old position data is useless)
  - DHCP (network setup)
  - NTP (time sync)
  - Log shipping (some systems use UDP syslog)

SRE INSIGHT:
  DNS uses UDP by default (small queries, speed matters).
  BUT DNS falls back to TCP for responses > 512 bytes
  (like zone transfers or DNSSEC responses).
```

### SRE Troubleshooting: TCP States

```bash
# When troubleshooting connection issues, TCP states matter:

ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
#    150 ESTAB         ← active connections (normal)
#     45 TIME-WAIT     ← recently closed, waiting for late packets
#     12 CLOSE-WAIT    ← remote closed, WE haven't closed yet (possible BUG)
#      3 SYN-SENT      ← trying to connect (possible network issue)

# CLOSE-WAIT piling up = YOUR application has a connection leak
# It received FIN from remote but hasn't called close()
# This is a common SRE interview follow-up!

# TIME-WAIT is NORMAL — it prevents old packets from confusing new connections
# But too many TIME-WAIT can exhaust ephemeral ports
# Tune: net.ipv4.tcp_tw_reuse = 1
```

### Interview Answer (30 seconds)

> "TCP is connection-oriented and guarantees reliable, ordered delivery through handshakes, sequence numbers, acknowledgments, and retransmission. UDP is connectionless and provides best-effort delivery with minimal overhead — faster but packets can be lost or arrive out of order. We use TCP for anything where data integrity matters like HTTP, SSH, and database connections. UDP is used where speed matters more than reliability — DNS queries, metrics shipping, and video streaming."

---

## 8. SWAP AREA

### Interview Question: "What is the swap area?"

### What Is Swap?

```
Swap is DISK SPACE used as an extension of RAM.

When physical RAM is full, the kernel moves less-used memory pages
from RAM to the swap area on disk. This is called "swapping out"
or "paging out."

When a process needs those pages again, the kernel moves them
back from disk to RAM ("swapping in" / "page fault").

ANALOGY:
  RAM = your desk (fast, limited space)
  Swap = a filing cabinet (slow, more space)
  
  When your desk is full, you move rarely-used papers to the cabinet.
  When you need them, you pull them back to your desk.
  Both are slower than having everything on your desk.
```

### How Swap Works at the Kernel Level

```
1. MEMORY PRESSURE
   - Kernel's memory management (mm) subsystem tracks free pages
   - When free memory drops below threshold, kswapd daemon wakes up

2. PAGE SELECTION
   - Kernel uses LRU (Least Recently Used) algorithm
   - Pages are on active/inactive lists
   - Frequently accessed pages stay in active list
   - Infrequently accessed pages move to inactive list
   - Pages from inactive list are candidates for swap

3. SWAP OUT
   - Kernel writes page contents to swap area on disk
   - Updates page table: "this page is now on disk at swap offset X"
   - Frees the RAM page for other use

4. SWAP IN (page fault)
   - Process accesses a swapped-out page
   - CPU triggers a PAGE FAULT (trap to kernel)
   - Kernel reads page from swap back to RAM
   - Updates page table to point to new RAM location
   - Process resumes (doesn't know anything happened)
```

### Swap Configuration

```bash
# Check swap:
free -h
#               total   used   free
# Mem:          16Gi    14Gi   2.0Gi
# Swap:         8.0Gi   2.0Gi  6.0Gi

swapon --show
# NAME       TYPE SIZE USED PRIO
# /dev/sda2  partition 8G  2G   -2
# /swapfile  file      4G  0B   -3

# Monitor swap activity:
vmstat 1
#  si  so    ← swap in / swap out (KB/s) — should be near 0
#   0   0    ← good, no swapping
# 500 200    ← bad, actively swapping = performance problem

# Swappiness (0-100): how aggressively kernel swaps
cat /proc/sys/vm/swappiness
# 60 (default)

# For databases (PostgreSQL, ClickHouse): set LOW
sudo sysctl vm.swappiness=10
# Databases need their data in RAM — swapping kills query performance

# For general servers: default 60 is usually fine
```

### OOM Killer — When Even Swap Isn't Enough

```bash
# When RAM AND swap are both exhausted:
# Linux kernel invokes the OOM (Out Of Memory) Killer
# It picks a process to kill based on oom_score

# Check OOM scores:
cat /proc/*/oom_score | sort -n | tail -5

# Protect critical processes from OOM:
echo -1000 > /proc/$(pidof postgresql)/oom_score_adj    # never kill this

# Check if OOM killed something:
dmesg | grep -i "oom\|out of memory\|killed process"
# This is a VERY common SRE investigation step
```

### Interview Answer (20 seconds)

> "Swap is disk space used as an extension of RAM. When physical memory is full, the kernel pages out infrequently-used memory pages to swap using LRU. When a process accesses a swapped page, it triggers a page fault and the kernel swaps it back in. For SRE, high swap activity (visible in `vmstat`) indicates memory pressure and performance degradation. Databases should have low swappiness since disk I/O is much slower than RAM."

---

## 9. LINUX KERNEL BOOT PROCESS

### Interview Question: "How does a Linux kernel load?"

### Boot Sequence — Step by Step

```
POWER ON → BIOS/UEFI → GRUB → KERNEL → INIT → SERVICES

Let's trace each step:

┌──────────────────────────────────────────────────────────┐
│ STEP 1: BIOS / UEFI (firmware)                          │
│                                                          │
│ - CPU starts executing from a hardcoded address          │
│ - POST (Power-On Self Test) — checks hardware            │
│ - Finds boot device (disk, USB, network)                 │
│ - BIOS: reads first 512 bytes (MBR) of boot disk        │
│   UEFI: reads EFI System Partition (ESP)                 │
│ - Hands control to the bootloader                        │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│ STEP 2: GRUB (GRand Unified Bootloader)                 │
│                                                          │
│ - GRUB Stage 1: tiny code in MBR, loads Stage 2         │
│ - GRUB Stage 2: full bootloader from /boot/grub/        │
│ - Displays boot menu (kernel versions, recovery options) │
│ - Reads config: /boot/grub/grub.cfg                     │
│ - Loads two files into memory:                           │
│   1. vmlinuz — the compressed kernel image               │
│   2. initramfs (initrd) — initial RAM filesystem         │
│ - Passes kernel command line parameters                  │
│   (root=, console=, quiet, etc.)                        │
│ - Jumps to kernel entry point                            │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│ STEP 3: KERNEL INITIALIZATION                            │
│                                                          │
│ - Decompresses itself (vmlinuz → vmlinux)                │
│ - Sets up memory management (page tables, zones)         │
│ - Initializes CPU scheduler                              │
│ - Detects and initializes hardware (PCI, USB, etc.)      │
│ - Mounts initramfs as temporary root filesystem          │
│ - initramfs contains:                                    │
│   - Essential drivers (disk, filesystem, RAID, LVM)      │
│   - Scripts to find and mount the REAL root filesystem   │
│ - Runs /init from initramfs                              │
│ - Switches root from initramfs to real root partition    │
│ - Starts PID 1 (init/systemd)                            │
└──────────────────────────────────────────────────────────┘
                          ↓
┌──────────────────────────────────────────────────────────┐
│ STEP 4: SYSTEMD (PID 1)                                  │
│                                                          │
│ - Reads unit files from /etc/systemd/system/             │
│ - Mounts filesystems (/etc/fstab)                        │
│ - Sets up networking                                     │
│ - Starts services in parallel based on dependencies      │
│ - Reaches "default target" (multi-user or graphical)     │
│ - System is ready                                        │
└──────────────────────────────────────────────────────────┘
```

### Why initramfs Exists

```
PROBLEM: The kernel needs filesystem drivers to mount the root
partition. But those drivers are ON the root partition!

SOLUTION: initramfs is a small temporary filesystem loaded
into RAM by GRUB. It contains just enough drivers and scripts
to find and mount the real root filesystem.

Think of it as a "bootstrap" filesystem:
  1. Kernel boots with initramfs as root
  2. initramfs loads disk drivers (SCSI, NVMe, RAID, LVM)
  3. initramfs mounts the real root partition
  4. Kernel switches root to the real filesystem
  5. initramfs is discarded

# See initramfs contents:
lsinitramfs /boot/initrd.img-$(uname -r)
# Shows drivers, scripts, and tools packed inside
```

### SRE Boot Troubleshooting

```bash
# Kernel parameters (view at runtime):
cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-5.15.0-91 root=/dev/sda1 ro quiet

# View boot messages:
dmesg | head -50                    # kernel ring buffer
journalctl -b                      # current boot log
journalctl -b -1                   # previous boot log

# Boot timing analysis:
systemd-analyze                    # total boot time
systemd-analyze blame             # which services took longest
systemd-analyze critical-chain    # dependency chain

# Common boot failures SRE encounters:
# 1. Kernel panic — usually hardware or driver issue
# 2. initramfs can't find root — wrong UUID in GRUB config
# 3. fsck fails — filesystem corruption
# 4. Service fails — broken config, missing dependency
```

### Interview Answer (30 seconds)

> "The boot sequence is: BIOS/UEFI performs POST and finds the boot device, then loads GRUB. GRUB loads the compressed kernel (vmlinuz) and initramfs into memory. The kernel decompresses, initializes hardware and memory management, then mounts initramfs as a temporary root to load essential drivers. Once the real root filesystem is mounted, the kernel starts systemd as PID 1, which starts all services in parallel. The initramfs is necessary because the kernel needs disk drivers to mount root, but those drivers are on the root disk — a chicken-and-egg problem."

---

## 10. `ls -l` AT THE KERNEL LEVEL

### Interview Question: "What happens in Linux, at a kernel level, when you type `ls -l`?"

### Full Walkthrough

```
You type: ls -l /home/sid/

Here's EVERYTHING that happens, from keystroke to output:

┌─ STEP 1: SHELL PROCESSING ──────────────────────────────┐
│                                                          │
│ 1. Bash reads your input from stdin (terminal)           │
│ 2. Parses "ls -l /home/sid/" into:                       │
│    command = "ls", args = ["-l", "/home/sid/"]           │
│ 3. Searches $PATH for "ls" binary                        │
│    /usr/bin/ls found                                     │
│ 4. Bash calls fork() → creates child process             │
│ 5. Child calls execve("/usr/bin/ls", args, env)          │
│    - Kernel loads the ls binary (ELF format)             │
│    - Maps it into memory                                 │
│    - Sets up stack, heap, args                           │
│    - Starts executing ls's main()                        │
│ 6. Parent (bash) calls waitpid() → waits for child      │
└──────────────────────────────────────────────────────────┘
                          ↓
┌─ STEP 2: ls READS THE DIRECTORY ─────────────────────────┐
│                                                          │
│ ls calls: openat(AT_FDCWD, "/home/sid/", O_DIRECTORY)    │
│                                                          │
│ Kernel resolves the path (VFS layer):                    │
│   1. Start at root inode (inode #2)                      │
│   2. Look up "home" → permission check → inode #100      │
│   3. Look up "sid" → permission check → inode #200       │
│   4. Open directory → returns file descriptor (fd 3)     │
│                                                          │
│ ls calls: getdents64(3, buffer, 1024)                    │
│   - Kernel reads directory entries from disk              │
│   - Returns array of: {inode_number, name, type}         │
│   - Example entries:                                     │
│     {inode: 301, name: "report.txt", type: regular}      │
│     {inode: 302, name: "scripts", type: directory}       │
└──────────────────────────────────────────────────────────┘
                          ↓
┌─ STEP 3: ls GETS FILE DETAILS (-l flag) ─────────────────┐
│                                                          │
│ For EACH file, ls calls: fstatat(fd, "report.txt", buf)  │
│                                                          │
│ This is a stat() system call. Kernel reads the INODE:    │
│   - File type & permissions (mode: 0100644 → -rw-r--r--)│
│   - Hard link count (nlink: 1)                           │
│   - Owner UID (1000) → ls calls getpwuid() → "sid"      │
│   - Group GID (1000) → ls calls getgrgid() → "sid"      │
│   - File size in bytes (1024)                            │
│   - Modification time (mtime: 2025-05-20 14:30)         │
│                                                          │
│ For symlinks: ls also calls readlink() to get target     │
└──────────────────────────────────────────────────────────┘
                          ↓
┌─ STEP 4: OUTPUT ─────────────────────────────────────────┐
│                                                          │
│ ls formats everything into columns:                      │
│ -rw-r--r-- 1 sid sid 1024 May 20 14:30 report.txt       │
│                                                          │
│ ls calls write(1, formatted_string, length)              │
│   fd 1 = stdout → terminal                              │
│   Kernel copies data to terminal's buffer                │
│   Terminal displays it on screen                         │
│                                                          │
│ ls calls close(3) → closes directory fd                  │
│ ls calls exit_group(0) → tells kernel it's done          │
│ Kernel sends SIGCHLD to parent (bash)                    │
│ Bash's waitpid() returns → bash shows prompt again       │
└──────────────────────────────────────────────────────────┘
```

### Verify With strace

```bash
# See the actual system calls:
strace ls -l /home/sid/ 2>&1 | head -30

# Key syscalls you'll see:
# execve("/usr/bin/ls", ["ls", "-l", "/home/sid/"], ...)
# openat(AT_FDCWD, "/home/sid/", O_RDONLY|O_DIRECTORY)
# getdents64(3, /* entries */, 32768)
# fstatat(3, "report.txt", {st_mode=S_IFREG|0644, st_size=1024, ...})
# fstatat(3, "scripts", {st_mode=S_IFDIR|0755, ...})
# write(1, "-rw-r--r-- 1 sid sid 1024 May 20 14:30 report.txt\n", 52)
# close(3)
# exit_group(0)

# Summarize syscalls:
strace -c ls -l /home/sid/
# % time     calls  syscall
# ------ --------- -------
#  35.00        15  fstatat
#  20.00         1  getdents64
#  10.00         1  openat
#   5.00         1  write
```

### Interview Answer (45 seconds)

> "When you type `ls -l`, bash forks a child process and the child calls execve to load the ls binary. ls opens the target directory with openat, then reads its entries with getdents64 — this returns filename-to-inode mappings. Because of the `-l` flag, ls calls fstatat on each file to read the inode metadata: permissions, owner, size, timestamps. It converts UIDs to names via getpwuid, formats everything, and writes to stdout. Finally, ls exits, the kernel sends SIGCHLD to bash, and bash's waitpid returns so it shows the prompt again. You can trace all of this with strace."

---

## 11. fork() RETURN VALUE

### Interview Question: "Which system call returns non-zero on success?"

### Answer: fork()

```c
// fork() creates a copy of the current process.
// It returns DIFFERENT values to parent and child:

pid_t pid = fork();

if (pid < 0) {
    // ERROR — fork failed (out of memory/PIDs)
}
else if (pid == 0) {
    // CHILD process — fork returned 0
    // (child gets 0 because it can find its parent via getppid())
}
else {
    // PARENT process — fork returned child's PID (positive number!)
    // Parent needs the child PID to wait() for it later
}

// So fork() returns:
//   - Negative: error
//   - 0: success (in child)
//   - Positive (child PID): success (in parent)
//
// The PARENT gets a NON-ZERO value on success!
// This is unique — most syscalls return 0 on success.
```

### Why This Design?

```
After fork(), there are TWO identical processes running the same code.
They need a way to know WHO they are:

Parent needs the child's PID:
  - To wait() for it later
  - To send it signals
  - To track it

Child already knows its own PID (getpid()) and parent's PID (getppid()).
So it doesn't need extra info — it gets 0.

This elegant design lets both processes continue from the same
point in the code but take different paths.
```

### Python Equivalent

```python
import os

pid = os.fork()

if pid == 0:
    # Child process
    print(f"I'm the child, my PID: {os.getpid()}, parent: {os.getppid()}")
    os._exit(0)
else:
    # Parent process
    print(f"I'm the parent, child PID: {pid}")
    os.waitpid(pid, 0)    # reap child (prevent zombie!)
```

### Interview Answer (15 seconds)

> "fork() returns a non-zero value on success — the parent receives the child's PID (a positive number), while the child receives 0. This lets both processes distinguish which one they are after the fork. On failure, it returns -1."

---

## 12. GARBAGE COLLECTION

### Interview Question: "How does a garbage collector work?"

### What Is Garbage Collection?

```
Garbage collection (GC) is AUTOMATIC MEMORY MANAGEMENT.
Instead of the programmer manually freeing memory (like in C),
the runtime finds and frees memory that's no longer used.

Languages with GC: Python, Java, Go, JavaScript, Ruby
Languages without: C, C++, Rust (uses ownership model instead)
```

### Python's Garbage Collection (3 Mechanisms)

```python
# Python uses THREE mechanisms:

# ─────────────────────────────────────────────
# MECHANISM 1: REFERENCE COUNTING (primary)
# ─────────────────────────────────────────────
# Every object has a counter tracking how many variables point to it.
# When counter hits 0, object is freed IMMEDIATELY.

a = "hello"          # "hello" refcount = 1
b = a                # "hello" refcount = 2
c = a                # "hello" refcount = 3
del b                # "hello" refcount = 2
del c                # "hello" refcount = 1
del a                # "hello" refcount = 0 → FREED immediately

import sys
x = [1, 2, 3]
print(sys.getrefcount(x))    # 2 (x + the getrefcount argument)

# Advantage: immediate cleanup, predictable
# Disadvantage: can't handle CIRCULAR references


# ─────────────────────────────────────────────
# MECHANISM 2: CYCLE DETECTOR (for circular refs)
# ─────────────────────────────────────────────
# When objects reference each other in a loop,
# refcount never reaches 0 even when nothing else uses them.

class Node:
    def __init__(self):
        self.next = None

a = Node()
b = Node()
a.next = b           # a → b
b.next = a           # b → a (CYCLE!)

del a                # a's refcount = 1 (b still points to it)
del b                # b's refcount = 1 (a still points to it)
# Neither refcount reaches 0! Memory leak without cycle detector.

# Python's gc module runs periodically and finds these cycles.
import gc
gc.collect()         # manually trigger cycle collection
gc.get_count()       # (gen0_count, gen1_count, gen2_count)


# ─────────────────────────────────────────────
# MECHANISM 3: GENERATIONAL COLLECTION
# ─────────────────────────────────────────────
# Based on the observation: most objects die young.
#
# Generation 0: new objects (collected most frequently)
# Generation 1: survived one collection
# Generation 2: long-lived objects (collected rarely)
#
# When Gen 0 fills up → collect Gen 0
# If objects survive → promote to Gen 1
# When Gen 1 fills up → collect Gen 0 AND Gen 1
# Gen 2 collected least often (expensive full scan)

gc.get_threshold()   # (700, 10, 10) — defaults
# Gen 0 triggers after 700 allocations - deallocations
# Gen 1 triggers every 10 Gen 0 collections
# Gen 2 triggers every 10 Gen 1 collections
```

### Java's Garbage Collection (Conceptual)

```
Java uses a more sophisticated approach:

HEAP MEMORY LAYOUT:
┌────────────────────────────────────────────────┐
│ Young Generation          │ Old Generation      │
│ ┌──────┬───────┬───────┐  │                     │
│ │ Eden │ S0    │ S1    │  │  Tenured Space      │
│ └──────┴───────┴───────┘  │                     │
└────────────────────────────────────────────────┘

1. New objects → Eden space
2. When Eden fills → Minor GC
   - Live objects copied to Survivor space (S0 or S1)
   - Dead objects freed
   - Fast! Only scans young generation
3. Objects that survive many Minor GCs → promoted to Old Generation
4. When Old fills → Major GC (Full GC)
   - Scans ENTIRE heap
   - SLOW — can cause "stop the world" pause
   - This is what causes Java application latency spikes!

SRE RELEVANCE:
  "Our Java service has periodic latency spikes every few minutes"
  → Likely Full GC pauses
  → Check with: jstat -gc PID
  → Tune with: -Xmx, -XX:+UseG1GC, etc.
```

### Go's Garbage Collection

```
Go uses a CONCURRENT, TRI-COLOR MARK-AND-SWEEP collector:

1. MARK PHASE (concurrent with application):
   - Start from "roots" (stack variables, globals)
   - Trace all reachable objects
   - Mark them as "alive"
   - Uses three colors: white (unknown), gray (found, not traced), black (done)

2. SWEEP PHASE:
   - Free all unmarked (white) objects

Key advantage: Go's GC runs CONCURRENTLY with the application.
Pause times are typically < 1ms (vs Java's potential 100ms+ pauses).

SRE relevance: Go services have more predictable latency than Java.
```

### Interview Answer (30 seconds)

> "A garbage collector automatically frees memory that's no longer reachable. Python uses reference counting as its primary mechanism — each object tracks how many references point to it, and when the count hits zero, it's freed immediately. For circular references, Python has a cycle detector that periodically scans for unreachable reference cycles. It uses generational collection, scanning newer objects more frequently since most objects die young. Java uses a generational heap with Minor GC for the young generation and Full GC for the old generation — Full GC can cause stop-the-world pauses, which is a common SRE troubleshooting scenario for latency spikes."

---

## QUICK REFERENCE: ALL 12 ANSWERS

```
Q: Zombie process?
A: Finished process waiting for parent to call wait(). Kill the PARENT, not the zombie.

Q: Hard vs soft link?
A: Hard = same inode, survives deletion. Soft = path pointer, breaks if target deleted.

Q: Inodes?
A: Store all file metadata except name. Can exhaust even with free disk (df -i).

Q: Signals?
A: Software interrupts. Kernel marks pending, delivers on return to user mode.

Q: Default kill signal?
A: SIGTERM (15). Graceful. Use SIGKILL (9) only as last resort.

Q: Terminate a process?
A: kill PID (SIGTERM first), then kill -9 PID if needed. Use pkill for by-name.

Q: TCP vs UDP?
A: TCP = reliable/ordered/connection. UDP = fast/unreliable/connectionless.

Q: Swap?
A: Disk-backed RAM extension. Kernel pages out inactive pages. vmstat shows activity.

Q: Kernel boot?
A: BIOS → GRUB loads kernel + initramfs → kernel init → mount root → systemd (PID 1).

Q: ls -l at kernel level?
A: fork→execve(ls)→openat(dir)→getdents64(entries)→fstatat(each file)→write(stdout)→exit.

Q: fork() non-zero success?
A: Parent gets child PID (positive). Child gets 0. Both on success.

Q: Garbage collection?
A: Auto memory management. Python: refcount + cycle detector + generational. Java: generational heap with GC pauses.
```

---

> **HOW TO STUDY THIS FILE**
>
> 1. Read each topic once — understand the concepts
> 2. Cover the "Interview Answer" and try to explain from memory
> 3. Practice saying each answer out loud in 30 seconds
> 4. For hands-on practice:
>    - Create a zombie process, find it with `ps`, fix it
>    - Create hard and soft links, delete the target, see the difference
>    - Run `strace ls -l` and trace the system calls
>    - Run `strace -e trace=signal kill -TERM $$` to see signal delivery
>    - Run `df -i` and `free -h` and `vmstat 1` on a real server
> 5. Re-read before each interview
