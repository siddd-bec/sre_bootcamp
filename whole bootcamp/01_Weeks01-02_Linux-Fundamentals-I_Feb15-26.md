# LINUX & SYSTEMS — FILE 01 of 48
## LINUX FUNDAMENTALS I
### Filesystem, Navigation, Commands, Pipes, Permissions, Processes
**Weeks 1-2 | Feb 15 - Feb 26, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY THIS MATTERS FOR SRE**
>
> Every production system at Apple, Google, Meta runs Linux.
> SRE interview question #1: "A server is slow. Walk me through debugging."
> This file gives you the foundation to answer confidently.
>
> After these 2 weeks you will:
> - Navigate any Linux filesystem blindfolded
> - Chain commands with pipes to solve real problems
> - Manage users, groups, and file permissions
> - Monitor and manage running processes
> - Write basic shell one-liners for automation

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Filesystem & Navigation | /, /etc, /var, /proc, cd, ls, find |
| 2 | File Operations | cp, mv, rm, mkdir, ln, tar, gzip, file descriptors |
| 3 | Text Processing I | cat, head, tail, wc, sort, uniq, cut, tr |
| 4 | Text Processing II | grep, sed basics, awk basics, pipes |
| 5 | Practice Lab I | Combined exercises with real log files |
| 6 | Users & Permissions | useradd, chmod, chown, umask, sudo |
| 7 | Processes I | ps, top, htop, kill, signals, /proc |
| 8 | Processes II | systemctl, journalctl, cron, background jobs |
| 9 | I/O & Redirection | stdin/stdout/stderr, >, >>, 2>&1, tee, xargs |
| 10 | Capstone | Full troubleshooting lab |

---

## DAY 1: FILESYSTEM & NAVIGATION
**Feb 15, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Linux filesystem hierarchy, everything is a file
> 00:20-01:10 Hands-on: navigation commands, find, locate, which
> 01:10-01:30 Drills: 10 rapid-fire exercises

### The Linux Filesystem Hierarchy

```bash
# Linux has ONE root: /
# Everything branches from here — no C: or D: drives

/           # root of entire filesystem
  /bin      # essential user commands (ls, cp, cat, grep)
  /sbin     # system admin commands (fdisk, iptables, reboot)
  /etc      # configuration files (ALL text, editable)
    /etc/passwd       # user accounts (one line per user)
    /etc/shadow       # encrypted passwords (root only)
    /etc/hosts        # local DNS overrides
    /etc/ssh/         # SSH server/client config
    /etc/systemd/     # service definitions
  /var      # variable data (changes during operation)
    /var/log/         # system logs (syslog, auth, kern)
    /var/lib/         # application state (databases, docker)
    /var/tmp/         # persistent temp files
  /home     # user home directories (/home/siddharth)
  /root     # root user home directory
  /tmp      # temporary files (cleared on reboot)
  /proc     # virtual filesystem — live kernel/process data
    /proc/cpuinfo     # CPU information
    /proc/meminfo     # memory stats
    /proc/1/          # info about process PID 1 (systemd)
  /sys      # virtual filesystem — hardware/driver info
  /dev      # device files (sda=disk, null, zero, random)
  /opt      # optional/third-party software
  /usr      # user programs and libraries
    /usr/bin/         # most user commands live here
    /usr/local/       # locally installed software
```

> **SRE INSIGHT: /proc IS YOUR BEST FRIEND**
>
> ```bash
> cat /proc/loadavg     # load average right now
> cat /proc/meminfo     # exact memory breakdown
> cat /proc/cpuinfo     # CPU model, cores, speed
> cat /proc/net/dev     # network interface stats
> cat /proc/1234/status # full details about process 1234
> ```
> Interview tip: "I check /proc when I don't have monitoring tools installed."

### Essential Navigation Commands

```bash
# Where am I?
pwd                          # print working directory

# Move around:
cd /var/log                  # absolute path
cd ..                        # parent directory
cd ~                         # home directory
cd -                         # previous directory (toggle)

# List contents:
ls -la                       # long format + hidden files (most useful!)
ls -lah                      # human-readable sizes
ls -lt                       # sort by modification time
ls -lS                       # sort by size

# Find files:
find / -name 'nginx.conf'              # find by name anywhere
find /var/log -name '*.log' -mtime -1  # logs modified in last 24h
find /tmp -size +100M                  # files larger than 100MB
find / -user root -perm -4000          # find SUID binaries (security!)

# Fast find:
locate nginx.conf           # instant (uses database)
updatedb                    # refresh locate database

# Where is a command?
which python3               # /usr/bin/python3
type ls                     # shows if aliased
whereis grep                # binary, source, man page
```

### DAY 1 EXERCISES

**EXERCISE 1: Explore the Filesystem**
Navigate to `/etc`, `/var/log`, `/proc`, `/dev`, `/tmp`, `/home` and list contents. For each: what kind of files are stored here?

**EXERCISE 2: Find Commands**
- a) Find all .conf files under /etc
- b) Find all files larger than 10MB in /var
- c) Find all files modified in the last hour under /tmp
- d) Find all directories named 'log' anywhere
- e) Find the nginx binary

**EXERCISE 3: /proc Exploration**
- a) How many CPU cores? `cat /proc/cpuinfo | grep processor | wc -l`
- b) Total memory? `grep MemTotal /proc/meminfo`
- c) Uptime in seconds? `cat /proc/uptime`
- d) What is PID 1? `cat /proc/1/comm`
- e) Count running PIDs: `ls /proc | grep -E '^[0-9]+$' | wc -l`

**DRILL: 10 Rapid-Fire (2 min each)**
1. Print current directory
2. Go to /var/log, list files sorted by size
3. Go back to home in one command
4. Find all .log files under /var modified today
5. Count files in /etc (non-recursive)
6. Find the bash binary location
7. Check CPU model from /proc
8. List hidden files in home directory
9. Find all empty files in /tmp
10. Display first 5 lines of /etc/passwd

---

## DAY 2: FILE OPERATIONS
**Feb 16, 2025 | 1.5 hrs**

### Core File Operations

```bash
# Copy:
cp file.txt backup.txt           # copy file
cp -r /etc/nginx /tmp/nginx-bak  # recursive
cp -p file.txt backup.txt        # preserve permissions

# Move/Rename:
mv old.txt new.txt               # rename
mv file.txt /tmp/                # move

# Remove:
rm file.txt                      # delete file
rm -r directory/                 # recursive
rm -rf /tmp/test/                # force (CAREFUL!)

# Directories:
mkdir -p /opt/app/logs/2025      # create full path

# Links:
ln -s /etc/nginx/nginx.conf ~/nginx.conf   # symbolic (shortcut)
ln /var/log/syslog /tmp/syslog-hard        # hard (same inode)

# Archives:
tar czf backup.tar.gz /var/log/  # create gzipped tar
tar xzf backup.tar.gz            # extract
tar tzf backup.tar.gz            # list contents
gzip largefile.log               # compress
gunzip largefile.log.gz          # decompress
zcat largefile.log.gz            # read without extracting
```

> **FILE DESCRIPTORS — INTERVIEW FAVORITE**
>
> Every process has 3 standard file descriptors:
> - 0 = stdin (input), 1 = stdout (output), 2 = stderr (errors)
>
> ```bash
> command > file.txt       # redirect stdout
> command 2> errors.txt    # redirect stderr
> command > out.txt 2>&1   # redirect BOTH to same file
> command &> both.txt      # shorthand (bash)
> command >> file.txt      # APPEND
> ```
> Interview: "What does 2>&1 mean?"
> Answer: "Redirect fd 2 (stderr) to wherever fd 1 (stdout) points."

### DAY 2 EXERCISES

**EXERCISE 1:** Create `/tmp/lab/{data,logs,config}`, 5 test files, copy, rename, symlink.

**EXERCISE 2:** Create tar.gz of /var/log, list without extracting, extract, compare sizes.

**EXERCISE 3:** Run `ls /etc /nonexistent` — redirect stdout only, stderr only, both, append mode.

---

## DAY 3: TEXT PROCESSING I
**Feb 17, 2025 | 1.5 hrs**

```bash
# View:
cat file.txt                    # entire file
head -20 / tail -50 file.txt   # first/last N lines
tail -f /var/log/syslog         # FOLLOW real-time (SRE essential!)
less file.txt                   # paginated

# Count:
wc -l / -w / -c file.txt       # lines / words / bytes

# Sort:
sort -rn file.txt               # reverse numeric
sort -t: -k3 -n /etc/passwd     # sort by field 3

# Unique + frequency:
sort file.txt | uniq -c | sort -rn  # frequency analysis!

# Extract columns:
cut -d: -f1 /etc/passwd         # usernames
cut -d' ' -f1,4 access.log      # IP and status

# Transform:
tr 'a-z' 'A-Z' < file.txt      # uppercase
tr -d '"' < file.txt            # delete chars
tr -s ' ' < file.txt            # squeeze spaces
```

> **SRE POWER MOVE: PIPE CHAINS**
> ```bash
> # Top 10 IPs hitting your web server:
> cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn | head -10
>
> # Count errors per hour:
> grep ERROR app.log | cut -d' ' -f1 | cut -dT -f2 | cut -d: -f1 | sort | uniq -c
> ```

### DAY 3 EXERCISES
**Log Analysis:** Count lines, extract IPs, top 5 IPs, requests per status, 500 error IPs.
**/etc/passwd Analysis:** Count users, extract usernames, bash users, sort by UID, UID > 1000.
**One-Liner Drills:** Unique words, longest line, extract emails, tabs to commas, lines 50-75.

---

## DAY 4: TEXT PROCESSING II — GREP, SED, AWK
**Feb 18, 2025 | 1.5 hrs**

### grep

```bash
grep 'ERROR' app.log              # basic match
grep -i 'error' app.log           # case-insensitive
grep -v 'DEBUG' app.log           # invert match
grep -c 'ERROR' app.log           # count
grep -n 'ERROR' app.log           # line numbers
grep -r 'password' /etc/          # recursive
grep -E 'ERROR|FATAL' app.log     # extended regex
grep -oE '[0-9]+ms' app.log       # extract matches only
grep -A3 -B2 'ERROR' app.log      # context lines
```

### sed

```bash
sed 's/old/new/g' file.txt           # replace all
sed -i 's/old/new/g' file.txt        # in-place edit
sed -i.bak 's/old/new/g' file.txt    # in-place with backup
sed '/DEBUG/d' app.log               # delete matching lines
sed -n '10,20p' file.txt             # print lines 10-20
```

### awk

```bash
awk '{print $1}' file.txt                    # first field
awk -F: '{print $1, $3}' /etc/passwd         # custom delimiter
awk '$3 > 1000' /etc/passwd                  # conditions
awk '{sum += $1} END {print sum}' nums.txt   # sum column
awk '{sum += $1; n++} END {print sum/n}'     # average
```

### DAY 4 EXERCISES
**grep:** Find IPs, count ERROR/WARN/INFO, extract timestamps, find slow requests, security audit.
**sed:** Replace http->https, delete comments, add prefix, extract between patterns, replace IPs.
**awk:** Print specific fields, average response time, count per status, top IP, sum bytes.

---

## DAY 5: PRACTICE LAB I
**Feb 19, 2025 | 1.5 hrs — COMBINE DAYS 1-4**

**LAB 1 (30 min):** Web log — unique IPs, top URLs, 404s, busiest hour, suspicious IPs, avg response time per URL.
**LAB 2 (30 min):** Health check — disk usage, largest files, memory, processes per user, services, connections.
**LAB 3 (30 min):** Log cleanup — files > 50MB, not modified 30 days, archive, verify, space saved.

---

## DAY 6: USERS & PERMISSIONS
**Feb 20, 2025 | 1.5 hrs**

```bash
# Permission values: r=4, w=2, x=1
# 755 = rwxr-xr-x (scripts), 644 = rw-r--r-- (configs), 600 = rw------- (keys)

chmod 755 script.sh              # set permissions
chown siddharth:sre-team file.txt
umask 022                        # default: files 644, dirs 755

# Special: SUID(4xxx) SGID(2xxx) Sticky(1xxx)
# Audit: find / -perm -4000 -type f   # find SUID binaries
```

### DAY 6 EXERCISES
Create users/groups, shared dirs with SGID, permission scenarios for SSH keys/web roots/secrets, security audit for SUID/world-writable/no-owner files.

---

## DAY 7: PROCESSES I
**Feb 21, 2025 | 1.5 hrs**

```bash
ps aux --sort=-%cpu | head -10  # top CPU consumers
top / htop                       # interactive monitor
pstree -p                        # hierarchy

# Signals:
kill PID          # SIGTERM (graceful)
kill -9 PID       # SIGKILL (force, last resort)
kill -HUP PID     # reload config
pkill -f 'python' # kill by pattern
```

> **Process States:** R=Running, S=Sleeping, D=Uninterruptible(disk), Z=Zombie(bug), T=Stopped.
> Many D = I/O bottleneck. Zombies = parent bug. High load + low CPU = iowait.

---

## DAY 8: PROCESSES II — SERVICES & SCHEDULING
**Feb 22, 2025 | 1.5 hrs**

```bash
systemctl status/start/stop/restart/enable nginx
systemctl list-units --type=service --state=failed
journalctl -u nginx -f              # follow logs
journalctl -p err --since today     # today's errors

# cron: min hour dom month dow command
# */5 * * * * /opt/health-check.sh  # every 5 min
# 0 2 * * * /opt/cleanup.sh         # daily 2 AM
```

---

## DAY 9: I/O & REDIRECTION MASTERY
**Feb 24, 2025 | 1.5 hrs**

```bash
command | tee output.log              # save AND display
find /tmp -name '*.log' | xargs rm   # xargs
cat servers.txt | xargs -P4 -I{} ssh {} uptime  # parallel!
diff <(sort f1) <(sort f2)           # process substitution
```

**SRE One-Liner Challenges:** Top 10 IPs, 5xx percentage, most FDs, disk monitor, listening ports, JSON parsing, config diff, /etc audit, pattern kill, parallel ping.

---

## DAY 10: CAPSTONE LAB
**Feb 25, 2025 | 1.5 hrs**

**Scenario: Slow Application Response**
1. System health (top, free, df, ss)
2. Process analysis (find app, check resources)
3. Log analysis (grep errors, access log patterns)
4. Root cause (CPU? Memory? Disk? Network?)
5. Fix and verify
6. Document: what, how, prevention

---

> **FILE 01 COMPLETE — LINUX FUNDAMENTALS I**
> Next: File 02 — Linux Fundamentals II (Weeks 3-4, Mar 1-12)
