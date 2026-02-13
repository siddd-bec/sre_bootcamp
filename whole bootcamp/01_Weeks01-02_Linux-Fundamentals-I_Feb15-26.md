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

```
In Linux, EVERYTHING starts from one root: /
There are no drive letters (no C: or D:). Just one tree.

/                    Root of the entire filesystem
├── bin/             Essential user commands (ls, cp, cat, grep)
├── sbin/            System admin commands (fdisk, iptables, reboot)
├── etc/             Configuration files — ALL plain text, editable
│   ├── passwd       User accounts (one line per user)
│   ├── shadow       Encrypted passwords (root only!)
│   ├── hosts        Local DNS overrides
│   ├── ssh/         SSH server and client config
│   ├── systemd/     Service unit files
│   ├── resolv.conf  DNS resolver configuration
│   └── fstab        Filesystem mount table (what mounts where at boot)
├── var/             Variable data — changes during operation
│   ├── log/         System and application logs
│   │   ├── syslog        General system log
│   │   ├── auth.log      Login/authentication events
│   │   ├── kern.log      Kernel messages
│   │   └── nginx/        Application-specific logs
│   ├── lib/         Application state (databases, docker images)
│   └── tmp/         Persistent temp files (survives reboot)
├── home/            User home directories (/home/siddharth)
├── root/            Root user's home directory
├── tmp/             Temporary files (cleared on reboot!)
├── proc/            Virtual filesystem — LIVE kernel and process data
│   ├── cpuinfo      CPU model, cores, speed
│   ├── meminfo      Memory breakdown (total, free, available, cached)
│   ├── loadavg      Load averages (1, 5, 15 min)
│   ├── uptime       Seconds since boot
│   ├── net/dev      Network interface statistics
│   └── [PID]/       Directory for each running process
│       ├── status   Process state, memory, threads
│       ├── cmdline  Full command that started process
│       ├── fd/      Open file descriptors
│       └── maps     Memory mappings
├── sys/             Virtual filesystem — hardware/driver info
├── dev/             Device files
│   ├── sda          First SCSI/SATA disk
│   ├── null         Discards everything written to it
│   ├── zero         Produces infinite zeros
│   └── random       Random number generator
├── opt/             Optional/third-party software
└── usr/             User programs and libraries
    ├── bin/         Most user commands live here
    ├── local/       Locally installed software
    └── share/       Shared data (man pages, docs)
```

> **SRE INSIGHT: /proc IS YOUR BEST FRIEND**
>
> /proc is not a real filesystem — it's a window into the kernel.
> No files are stored on disk. Everything is generated live.
>
> ```bash
> cat /proc/loadavg     # load averages + running/total processes
> cat /proc/meminfo     # exact memory breakdown (100+ lines)
> cat /proc/cpuinfo     # CPU model, cores, cache, flags
> cat /proc/net/dev     # network bytes/packets/errors per interface
> cat /proc/diskstats   # disk I/O statistics
> cat /proc/1/status    # full details about PID 1 (systemd)
> cat /proc/1/cmdline   # what command started PID 1
> ls /proc/1/fd         # all open file descriptors for PID 1
> ```
>
> **Interview tip:** "When I don't have monitoring tools installed,
> I go straight to /proc. It always has what I need."

### Essential Navigation Commands

```bash
# === WHERE AM I? ===
pwd                              # print working directory
# Output: /home/siddharth

# === MOVING AROUND ===
cd /var/log                      # go to absolute path
cd ..                            # go up one level (parent directory)
cd ../..                         # go up two levels
cd ~                             # go to home directory (/home/siddharth)
cd -                             # go to PREVIOUS directory (toggle back and forth)
cd                               # same as cd ~ (go home)

# === LISTING FILES ===
ls                               # basic listing
ls -l                            # long format (permissions, owner, size, date)
ls -la                           # long + hidden files (files starting with .)
ls -lah                          # long + hidden + human-readable sizes (1K, 2M, 3G)
ls -lt                           # sort by modification time (newest first)
ls -ltr                          # sort by time, reversed (oldest first)
ls -lS                           # sort by size (largest first)
ls -li                           # show inode numbers

# READING ls -l OUTPUT:
# -rw-r--r-- 1 siddharth sre-team 4096 Feb 15 10:30 config.yaml
# │          │ │          │        │    │             │
# │          │ │          │        │    │             └── filename
# │          │ │          │        │    └── modification date
# │          │ │          │        └── size in bytes
# │          │ │          └── group owner
# │          │ └── user owner
# │          └── hard link count
# └── file type + permissions (- = file, d = directory, l = symlink)
```

### Finding Files

```bash
# === find — the powerful file searcher ===
# Syntax: find [where] [conditions] [actions]

# By name:
find / -name "nginx.conf"                  # exact name, search everywhere
find /etc -name "*.conf"                   # all .conf files under /etc
find /var/log -name "*.log"                # all log files under /var/log
find / -iname "readme*"                    # case-insensitive

# By time:
find /var/log -mtime -1                    # modified in last 24 hours
find /tmp -mtime +7                        # modified more than 7 days ago
find /var/log -mmin -60                    # modified in last 60 minutes
find /etc -newer /etc/passwd               # newer than passwd file

# By size:
find / -size +100M                         # files larger than 100MB
find /var/log -size +1G                    # files larger than 1GB
find /tmp -empty                           # empty files and directories

# By type:
find /var -type f -name "*.log"            # files only (not directories)
find /etc -type d -name "nginx*"           # directories only
find /dev -type l                          # symbolic links

# By permission:
find / -perm -4000 -type f                 # SUID binaries (security audit!)
find / -perm -o+w -type f                  # world-writable files (security!)
find /home -perm 777                       # full open permissions (bad!)

# By owner:
find / -user root -perm -4000             # root-owned SUID files
find /tmp -nouser                          # files with no valid owner

# Combined with actions:
find /tmp -name "*.tmp" -mtime +7 -delete           # delete old temp files
find /var/log -name "*.log" -size +100M -exec ls -lh {} \;  # list large logs
find /etc -name "*.conf" -exec grep -l "password" {} \;      # configs with passwords

# === Quick alternatives ===
locate nginx.conf             # instant search (uses database, run updatedb to refresh)
which python3                 # finds command in PATH → /usr/bin/python3
type ls                       # shows if command is alias, builtin, or file
whereis grep                  # finds binary, source, and man page locations
```

### DAY 1 EXERCISES

> **EXERCISE 1: Filesystem Exploration**
>
> Navigate to each directory and describe what you find:
> a) `cd /etc && ls | head -20` — what kind of files?
> b) `cd /var/log && ls -lt | head -10` — which logs are freshest?
> c) `cd /proc && ls` — what are all the numbered directories?
> d) `cd /dev && ls` — what do null, zero, random do?
> e) `cd /tmp && ls -la` — who created these files?
> f) `cd /home && ls -la` — which users have home directories?

> **EXERCISE 2: Find Commands Practice**
>
> a) Find all .conf files under /etc
> b) Find all files larger than 10MB anywhere in /var
> c) Find all files modified in the last hour under /tmp
> d) Find all directories named "log" anywhere on the system
> e) Find the nginx binary (or any installed service binary)
> f) Find all empty files in /tmp
> g) Find all SUID binaries (security audit!): `find / -perm -4000 -type f 2>/dev/null`

> **EXERCISE 3: /proc Deep Dive**
>
> Answer these using ONLY /proc:
> a) How many CPU cores? `grep -c processor /proc/cpuinfo`
> b) Total physical memory? `grep MemTotal /proc/meminfo`
> c) How long since last reboot? `cat /proc/uptime` (first number = seconds)
> d) What is PID 1? `cat /proc/1/comm`
> e) Count running processes: `ls -d /proc/[0-9]* | wc -l`
> f) What's the load average? `cat /proc/loadavg`
> g) How much swap is in use? `grep SwapTotal /proc/meminfo`
> h) Network bytes received on each interface? `cat /proc/net/dev`

> **DRILL: 10 Rapid-Fire Navigation (2 min each)**
>
> 1. Print current directory
> 2. Go to /var/log, list files sorted by size (largest first)
> 3. Go back to home directory in one command
> 4. Find all .log files under /var modified in the last day
> 5. Count files in /etc (non-recursive): `ls -1 /etc | wc -l`
> 6. Find the bash binary location
> 7. Check CPU model from /proc
> 8. List hidden files in your home directory
> 9. Find all empty files in /tmp
> 10. Display first 5 lines of /etc/passwd

---

## DAY 2: FILE OPERATIONS
**Feb 16, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: copy, move, remove, links, archives
> 00:20-01:10 Hands-on: file manipulation exercises
> 01:10-01:30 Drills: rapid-fire file operations

### Core File Operations

```bash
# === COPY ===
cp file.txt backup.txt                    # copy file
cp -r /etc/nginx /tmp/nginx-backup        # copy directory recursively
cp -p file.txt backup.txt                 # preserve permissions, timestamps
cp -a /src/ /dst/                         # archive mode (preserves everything)
cp -i file.txt backup.txt                 # interactive (confirm overwrite)

# === MOVE / RENAME ===
mv old_name.txt new_name.txt              # rename
mv file.txt /tmp/                         # move to another directory
mv file.txt /tmp/new_name.txt             # move AND rename
mv -i file.txt /tmp/                      # interactive (confirm overwrite)

# === REMOVE ===
rm file.txt                               # delete file
rm -i file.txt                            # interactive (confirm)
rm -r directory/                          # delete directory recursively
rm -rf /tmp/test/                         # force, no confirm (CAREFUL!)
rmdir empty_directory/                    # only deletes EMPTY directories

# SRE SAFETY: Never run rm -rf with variables unless validated:
# BAD:  rm -rf $DIR/     (if DIR is empty, this becomes rm -rf /)
# GOOD: rm -rf "${DIR:?ERROR: DIR not set}/"

# === CREATE DIRECTORIES ===
mkdir mydir                               # create single directory
mkdir -p /opt/app/logs/2025/02            # create full path (all parents)

# === TOUCH ===
touch newfile.txt                         # create empty file (or update timestamp)
touch -t 202501151030 file.txt            # set specific timestamp
```

### Links — Hard and Symbolic

```bash
# SYMBOLIC LINK (symlink) — like a shortcut/pointer
ln -s /etc/nginx/nginx.conf ~/nginx.conf
# ~/nginx.conf → /etc/nginx/nginx.conf
# If target deleted, symlink BREAKS (dangling link)
# Can cross filesystems, can link to directories

# SRE USE CASE — Deployment symlinks:
# /opt/app/current → /opt/app/release-v2.3.1
# Deploy new version by changing the symlink. Instant rollback!

# HARD LINK — another name for the SAME data (same inode)
ln /var/log/syslog /tmp/syslog-hardlink
# Both names point to same data blocks on disk
# Data only deleted when ALL hard links are removed
# Cannot cross filesystems, cannot link to directories

# Check links:
ls -li file.txt                # -i shows inode number
readlink -f symlink.txt        # shows where symlink points
```

### Archives and Compression

```bash
# === tar (Tape ARchive) — the standard ===
# c=create, x=extract, t=list, z=gzip, j=bzip2, v=verbose, f=file

tar czf backup.tar.gz /var/log/           # create gzipped archive
tar czf backup.tar.gz -C /opt/app .       # archive contents of directory
tar xzf backup.tar.gz                     # extract
tar xzf backup.tar.gz -C /tmp/            # extract to specific directory
tar tzf backup.tar.gz                     # list contents without extracting
tar czf backup.tar.gz --exclude='*.tmp' /var/log/  # exclude pattern

# === Compression tools ===
gzip largefile.log                        # compress → largefile.log.gz (original deleted)
gunzip largefile.log.gz                   # decompress
zcat largefile.log.gz                     # read compressed without extracting
zgrep "ERROR" largefile.log.gz            # grep inside compressed file!

# Size comparison:
ls -lh file.log file.log.gz
# -rw-r--r-- 1 root root 150M file.log
# -rw-r--r-- 1 root root  12M file.log.gz    ← 92% compression on logs!
```

### File Descriptors — Interview Favorite

```
Every running process has 3 standard file descriptors (FDs):

  FD 0 = stdin  (standard input)  — where the process reads from
  FD 1 = stdout (standard output) — where the process writes normal output
  FD 2 = stderr (standard error)  — where the process writes error messages

  ┌─────────┐
  │ Process │
  │         │──→ FD 1 (stdout) → terminal (or file)
  │         │──→ FD 2 (stderr) → terminal (or file)
  │         │←── FD 0 (stdin)  ← keyboard (or file/pipe)
  └─────────┘

WHY THIS MATTERS:
  Normal output and errors go to DIFFERENT streams.
  You can redirect them independently.
  This is how SRE scripts capture errors separately from output.
```

```bash
# Redirect stdout only:
ls /etc > output.txt               # stdout to file (overwrites)
ls /etc >> output.txt              # stdout to file (appends)

# Redirect stderr only:
ls /nonexistent 2> errors.txt      # stderr to file

# Redirect BOTH to same file:
ls /etc /nonexistent > all.txt 2>&1    # stdout to file, stderr follows
ls /etc /nonexistent &> all.txt        # shorthand (bash only)

# Redirect to different files:
ls /etc /nonexistent > out.txt 2> err.txt

# Discard output:
command > /dev/null 2>&1           # silence everything

# INTERVIEW QUESTION: "What does 2>&1 mean?"
# ANSWER: "Redirect file descriptor 2 (stderr) to wherever
#          file descriptor 1 (stdout) currently points."
```

### DAY 2 EXERCISES

> **EXERCISE 1: File Operations Lab**
>
> a) Create directory structure: `/tmp/lab/{data,logs,config,backup}`
> b) Create 5 files in data/ with some content (use echo or printf)
> c) Copy all files from data/ to backup/
> d) Rename one file in backup/
> e) Create a symlink from config/current → data/config.txt
> f) Verify the symlink works: `cat config/current`
> g) Delete the target file — what happens to the symlink?

> **EXERCISE 2: Archives**
>
> a) Create a tar.gz of /var/log (or /tmp/lab if no access)
> b) List contents without extracting
> c) Extract to a different directory
> d) Compare sizes: original vs compressed
> e) Use zgrep to search inside a .gz file without extracting

> **EXERCISE 3: Redirection Mastery**
>
> Run `ls /etc /nonexistent` and:
> a) Redirect stdout only to a file
> b) Redirect stderr only to a file
> c) Redirect both to the same file
> d) Redirect stdout to one file and stderr to another
> e) Redirect both to /dev/null (silence completely)
> f) Append stdout to an existing file

> **DRILL: 10 File Operation Challenges (2 min each)**
>
> 1. Create a nested directory path with one command
> 2. Copy a directory recursively preserving permissions
> 3. Create a symbolic link
> 4. Check what a symlink points to
> 5. Create a tar.gz archive
> 6. Extract a tar.gz to a specific directory
> 7. Redirect stderr to a file
> 8. Explain what 2>&1 does
> 9. Read a .gz file without extracting it
> 10. Safely delete a directory (what flag combination?)

---

## DAY 3: TEXT PROCESSING I
**Feb 17, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: viewing, counting, sorting, extracting
> 00:20-01:10 Hands-on: process real log data
> 01:10-01:30 Drills: one-liner challenges

### Viewing Files

```bash
cat file.txt                     # display entire file
cat -n file.txt                  # with line numbers
head -20 file.txt                # first 20 lines
head -1 file.txt                 # just the first line (header)
tail -50 file.txt                # last 50 lines
tail -f /var/log/syslog          # FOLLOW in real-time (Ctrl+C to stop)
tail -f /var/log/syslog | grep ERROR   # follow only errors
less file.txt                    # paginated viewing (q to quit)
                                 # / to search, n for next match
                                 # G to go to end, g to go to start
```

> **tail -f IS THE SRE's LIVE VIEW**
>
> During an incident, `tail -f` is often the first thing you run.
> It shows new log lines as they're written in real-time.
> Combine with grep to filter for just what matters:
> ```bash
> tail -f /var/log/app.log | grep -E "ERROR|FATAL|CRITICAL"
> ```

### Counting

```bash
wc -l file.txt                   # count lines
wc -w file.txt                   # count words
wc -c file.txt                   # count bytes
wc -m file.txt                   # count characters
wc -l /var/log/*.log             # count lines in multiple files

# SRE: How many errors today?
grep -c "ERROR" /var/log/app.log
```

### Sorting

```bash
sort file.txt                    # alphabetical sort
sort -r file.txt                 # reverse sort
sort -n file.txt                 # numeric sort (1, 2, 10 not 1, 10, 2)
sort -rn file.txt                # reverse numeric (largest first)
sort -u file.txt                 # sort + remove duplicates
sort -k2 file.txt                # sort by 2nd field
sort -t: -k3 -n /etc/passwd      # sort by field 3 (UID), delimiter :
sort -h                          # human-readable numbers (1K, 2M, 3G)
```

### Unique and Frequency

```bash
# uniq only removes ADJACENT duplicates — ALWAYS sort first!
sort file.txt | uniq              # remove duplicates
sort file.txt | uniq -c           # count occurrences
sort file.txt | uniq -c | sort -rn    # frequency analysis (most common first)
sort file.txt | uniq -d           # show ONLY duplicates

# THE SRE POWER PATTERN — frequency analysis:
# What it does: sort → count adjacent → sort by count descending
# Example: Top 10 IPs hitting your server:
cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn | head -10
```

### Extracting Columns

```bash
cut -d: -f1 /etc/passwd           # extract field 1 (username), delimiter :
cut -d: -f1,3 /etc/passwd         # fields 1 and 3 (username, UID)
cut -d' ' -f1 access.log          # first field (IP address), delimiter space
cut -d' ' -f1,9 access.log        # IP and status code
cut -c1-10 file.txt               # first 10 characters of each line
```

### Transforming Text

```bash
tr 'a-z' 'A-Z' < file.txt        # lowercase to uppercase
tr 'A-Z' 'a-z' < file.txt        # uppercase to lowercase
tr -d '"' < file.txt              # delete all quote characters
tr -d '\r' < windows.txt          # remove Windows carriage returns
tr -s ' ' < file.txt              # squeeze multiple spaces into one
tr '\t' ',' < file.txt            # tabs to commas (TSV → CSV)
```

### Pipes — Chaining Commands

```bash
# The pipe | sends stdout of one command into stdin of the next.
# This is the CORE of Linux text processing.

# Count unique usernames logged in:
who | awk '{print $1}' | sort | uniq | wc -l

# Top 10 IPs in an access log:
cat access.log | cut -d' ' -f1 | sort | uniq -c | sort -rn | head -10

# Count errors per hour:
grep "ERROR" app.log | cut -d' ' -f1,2 | cut -d: -f1 | sort | uniq -c

# Find largest files in /var:
du -ah /var 2>/dev/null | sort -rh | head -20

# Count requests per HTTP status code:
awk '{print $9}' access.log | sort | uniq -c | sort -rn
```

### DAY 3 EXERCISES

> **EXERCISE 1: Log File Analysis**
>
> Create a sample access.log (or use a real one) and:
> a) Count total lines
> b) Extract all IP addresses (first field)
> c) Find the top 5 most frequent IPs
> d) Count requests per HTTP status code
> e) Find all lines with status 500
> f) Count 500 errors per IP address

> **EXERCISE 2: /etc/passwd Analysis**
>
> Using /etc/passwd:
> a) Count total users
> b) Extract just usernames (field 1)
> c) Find users who use /bin/bash as shell (field 7)
> d) Sort users by UID (field 3, numeric)
> e) Find users with UID > 1000

> **EXERCISE 3: One-Liner Challenges**
>
> a) Count unique words in a text file
> b) Find the longest line in a file: `awk '{print length, $0}' file | sort -rn | head -1`
> c) Extract email addresses from a file (grep -oE pattern)
> d) Convert tabs to commas in a file
> e) Show lines 50-75 of a file: `sed -n '50,75p' file`

> **DRILL: 10 Text Processing Challenges (2 min each)**
>
> 1. Show first 5 lines of a file
> 2. Follow a log file in real-time
> 3. Count lines in a file
> 4. Sort a file numerically in reverse
> 5. Get frequency count of repeated lines
> 6. Extract the 3rd field (delimiter is a space)
> 7. Convert lowercase to uppercase
> 8. Delete all digits from a string
> 9. Squeeze multiple spaces into one
> 10. Chain 3 commands with pipes

---

## DAY 4: TEXT PROCESSING II — GREP, SED, AWK
**Feb 18, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: grep patterns, sed substitution, awk fields
> 00:20-01:10 Hands-on: real log processing
> 01:10-01:30 Drills: pattern matching challenges

### grep — Search Inside Files

```bash
# Basic search:
grep "ERROR" app.log                     # lines containing ERROR
grep -i "error" app.log                  # case-insensitive
grep -v "DEBUG" app.log                  # INVERT — lines WITHOUT DEBUG
grep -c "ERROR" app.log                  # COUNT matching lines
grep -n "ERROR" app.log                  # show LINE NUMBERS

# Recursive search:
grep -r "password" /etc/                 # search all files under /etc
grep -rl "password" /etc/                # just filenames (not content)

# Extended regex:
grep -E "ERROR|FATAL|CRITICAL" app.log   # multiple patterns (OR)
grep -E "^2025-02-15" app.log            # lines STARTING with date
grep -E "timeout$" app.log               # lines ENDING with timeout

# Extract matches:
grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+" app.log   # extract IPs
grep -oE "[0-9]+ms" app.log              # extract response times

# Context lines:
grep -A 3 "ERROR" app.log               # 3 lines AFTER each match
grep -B 2 "ERROR" app.log               # 2 lines BEFORE each match
grep -C 2 "ERROR" app.log               # 2 lines BEFORE and AFTER

# Multiple patterns from file:
grep -f patterns.txt app.log             # search for all patterns in file

# SRE ESSENTIALS:
grep -E "ERROR|WARN" app.log | tail -20                    # recent errors
grep -c "ERROR" /var/log/app.log                           # error count
grep -r "TODO\|FIXME\|HACK" /opt/app/src/                  # code audit
zgrep "ERROR" /var/log/app.log.1.gz                        # search compressed!
```

### sed — Stream Editor

```bash
# SUBSTITUTION (the most common use):
sed 's/old/new/' file.txt                # replace FIRST occurrence per line
sed 's/old/new/g' file.txt               # replace ALL occurrences per line
sed 's/old/new/gi' file.txt              # case-insensitive, all occurrences

# In-place editing:
sed -i 's/old/new/g' file.txt            # modify the actual file
sed -i.bak 's/old/new/g' file.txt        # modify + keep backup (.bak)

# Delete lines:
sed '/DEBUG/d' app.log                   # delete lines matching pattern
sed '/^$/d' file.txt                     # delete empty lines
sed '/^#/d' config.txt                   # delete comment lines

# Print specific lines:
sed -n '10,20p' file.txt                 # print lines 10-20 only
sed -n '1p' file.txt                     # print first line only

# Multiple operations:
sed -e 's/foo/bar/g' -e '/^#/d' file.txt

# SRE USE CASES:
sed -i 's/listen 80/listen 8080/' nginx.conf               # change port
sed -i 's/DEBUG/INFO/' /etc/app/config.yaml                 # change log level
sed 's/password=.*/password=***REDACTED***/' log.txt        # redact passwords
sed -n '/2025-02-15 14:00/,/2025-02-15 15:00/p' app.log   # time range
```

### awk — Column Processing

```bash
# awk splits each line into fields: $1, $2, $3... ($0 = whole line)

# Print specific fields:
awk '{print $1}' file.txt                        # first field
awk '{print $1, $3}' file.txt                    # first and third
awk '{print $NF}' file.txt                       # LAST field
awk '{print NR, $0}' file.txt                    # line number + line

# Custom field delimiter:
awk -F: '{print $1, $3}' /etc/passwd             # username, UID

# Conditions (filtering):
awk '$9 == 500' access.log                       # lines where field 9 = 500
awk '$9 >= 400' access.log                       # status code >= 400
awk 'length > 100' file.txt                      # lines longer than 100 chars
awk '$3 > 1000' data.txt                         # field 3 greater than 1000

# Math:
awk '{sum += $1} END {print sum}' numbers.txt                    # sum a column
awk '{sum += $1; n++} END {print sum/n}' numbers.txt             # average
awk '{sum += $10} END {print sum/NR}' access.log                 # avg response size
awk 'BEGIN {max=0} $10>max {max=$10} END {print max}' access.log # max value

# Formatted output:
awk '{printf "%-15s %5d\n", $1, $3}' data.txt    # formatted columns

# SRE POWER MOVES:
# Requests per second:
awk '{print $4}' access.log | cut -d: -f1-3 | sort | uniq -c | sort -rn | head

# Average response time:
awk '{sum += $NF; n++} END {printf "Avg: %.2f ms\n", sum/n}' access.log

# Top 10 slowest requests:
awk '{print $NF, $7}' access.log | sort -rn | head -10

# Bytes transferred per IP:
awk '{bytes[$1] += $10} END {for (ip in bytes) print bytes[ip], ip}' access.log | sort -rn | head
```

### DAY 4 EXERCISES

> **EXERCISE 1: grep Mastery**
>
> Using a log file (create one or use /var/log/syslog):
> a) Find all lines with "error" (case-insensitive)
> b) Count errors vs warnings vs info messages
> c) Extract all IP addresses from the log
> d) Find slow requests (response time > 1000ms)
> e) Search recursively in /etc for "password" references
> f) Show 3 lines of context around each error

> **EXERCISE 2: sed Practice**
>
> a) Replace all "http://" with "https://" in a config file
> b) Delete all comment lines (starting with #)
> c) Delete all empty lines
> d) Add a prefix "LOG: " to every line
> e) Extract lines between two patterns (time range)
> f) Redact all IP addresses: `sed -E 's/[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[REDACTED]/g'`

> **EXERCISE 3: awk Processing**
>
> Using an access log:
> a) Print only the IP and URL fields
> b) Calculate average response time
> c) Count requests per status code
> d) Find the IP with most requests
> e) Sum total bytes transferred
> f) Find top 10 slowest URLs

> **DRILL: 10 Pattern Matching Challenges (2 min each)**
>
> 1. grep for lines containing "ERROR" with line numbers
> 2. grep for lines NOT containing "DEBUG"
> 3. Count how many times "timeout" appears in a log
> 4. Replace all tabs with spaces using sed
> 5. Delete the first line of a file with sed
> 6. Print only the 2nd field with awk
> 7. Sum a column of numbers with awk
> 8. Filter lines where field 3 > 100 with awk
> 9. Extract emails with grep -oE
> 10. Chain grep | awk | sort | head for top-N analysis

---

## DAY 5: PRACTICE LAB I
**Feb 19, 2025 | 1.5 hrs — COMBINE DAYS 1-4**

> This is pure hands-on practice. Create sample log files or
> use real ones from /var/log. Build your muscle memory.

> **LAB 1: Web Server Log Analysis (30 min)**
>
> Using an access log:
> a) Total number of requests
> b) Unique IP addresses
> c) Top 10 IPs by request count
> d) Top 10 most requested URLs
> e) All 404 errors with URLs
> f) Busiest hour of the day
> g) Average response time per URL
> h) All requests from a specific IP

> **LAB 2: System Health Check (30 min)**
>
> Using Linux commands:
> a) Disk usage per partition (find anything > 80%)
> b) Find the 10 largest files on the system
> c) Memory usage (used, free, available, cached)
> d) Number of processes per user
> e) Which services are running?
> f) Number of network connections by state
> g) Current load average — is it concerning?

> **LAB 3: Log Cleanup Script (30 min)**
>
> Write a series of commands (or a script) that:
> a) Find all .log files larger than 50MB in /var/log
> b) Find all .log files not modified in 30 days
> c) Create a tar.gz archive of old logs
> d) Verify the archive is valid
> e) Calculate total space that would be saved
> f) (Bonus) Write this as a reusable script

---

## DAY 6: USERS & PERMISSIONS
**Feb 20, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: users, groups, permission model, special permissions
> 00:20-01:10 Hands-on: create users, set permissions, security audit
> 01:10-01:30 Drills: permission challenges

### Users and Groups

```bash
# Every file has an OWNER (user) and a GROUP.
# Every process runs AS a user.
# Permissions control who can read, write, execute.

# User management:
useradd -m -s /bin/bash deploy               # create user with home dir and shell
usermod -aG docker deploy                    # add user to group (without removing others)
userdel -r olduser                           # delete user + home directory
passwd deploy                                # set/change password

# Group management:
groupadd sre-team                            # create group
groupadd -g 1500 sre-team                    # with specific GID
usermod -aG sre-team siddharth               # add user to group

# View:
id siddharth                                 # uid, gid, groups
groups siddharth                             # group memberships
cat /etc/passwd | grep siddharth             # user entry
cat /etc/group | grep sre-team               # group entry
who                                          # currently logged in
last                                         # login history
```

### Permission Model

```
Every file has permissions for three categories:
  OWNER (u)   GROUP (g)   OTHERS (o)

Each category can have:
  r (read)    = 4
  w (write)   = 2
  x (execute) = 1

Example: -rwxr-xr-- = 754
  Owner: rwx = 4+2+1 = 7 (read + write + execute)
  Group: r-x = 4+0+1 = 5 (read + execute)
  Other: r-- = 4+0+0 = 4 (read only)

COMMON PERMISSIONS:
  755 = rwxr-xr-x  → scripts, executables, directories
  644 = rw-r--r--  → config files, regular files
  600 = rw-------  → SSH keys, secrets (OWNER ONLY)
  700 = rwx------  → private directories, .ssh/
  775 = rwxrwxr-x  → shared group directories
  666 = rw-rw-rw-  → (BAD! everyone can write)
  777 = rwxrwxrwx  → (VERY BAD! never use in production)
```

```bash
# Change permissions:
chmod 755 script.sh                          # numeric (absolute)
chmod u+x script.sh                          # add execute for owner
chmod g+w file.txt                           # add write for group
chmod o-r file.txt                           # remove read for others
chmod -R 755 /opt/app/                       # recursive

# Change ownership:
chown deploy:sre-team file.txt               # change owner AND group
chown deploy file.txt                        # change owner only
chown :sre-team file.txt                     # change group only
chown -R deploy:deploy /opt/app/             # recursive

# Default permissions:
umask                                        # show current umask
umask 022                                    # default: files=644, dirs=755
umask 077                                    # restrictive: files=600, dirs=700
# umask SUBTRACTS from maximum (666 for files, 777 for dirs)
```

### Special Permissions

```bash
# SUID (Set User ID) — execute as file OWNER (not as yourself)
# Number: 4xxx
chmod u+s /usr/bin/passwd                    # passwd runs as root
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root ...                # note the 's' in owner execute

# SGID (Set Group ID) — new files inherit directory's group
# Number: 2xxx
chmod g+s /opt/shared/                       # new files get directory's group
# Great for shared team directories

# Sticky Bit — only owner can delete their files
# Number: 1xxx
chmod +t /tmp/                               # /tmp uses this!
ls -ld /tmp
# drwxrwxrwt ...                             # note the 't' at the end
# Everyone can write to /tmp, but only owner can delete their own files

# SUID/SGID SECURITY AUDIT (common interview question!):
find / -perm -4000 -type f 2>/dev/null       # find all SUID binaries
find / -perm -2000 -type f 2>/dev/null       # find all SGID binaries
find / -perm -o+w -type f 2>/dev/null        # world-writable files
```

### sudo — Run as Root

```bash
sudo command                                 # run single command as root
sudo -u deploy command                       # run as specific user
sudo -i                                      # open root shell
sudo !!                                      # re-run last command as root

# sudoers config (/etc/sudoers — ONLY edit with visudo!):
# siddharth ALL=(ALL) NOPASSWD: ALL         # full sudo, no password
# %sre-team ALL=(ALL) /usr/bin/systemctl    # group can only run systemctl
```

### DAY 6 EXERCISES

> **EXERCISE 1: User and Group Setup**
>
> a) Create a group "sre-team"
> b) Create users: deploy, monitor, backup
> c) Add all to sre-team group
> d) Create shared directory /opt/sre-shared with SGID
> e) Verify: create files as different users, check group ownership

> **EXERCISE 2: Permission Scenarios**
>
> Set correct permissions for:
> a) SSH private key (~/.ssh/id_ed25519) → 600
> b) SSH directory (~/.ssh/) → 700
> c) Web root (/var/www/html/) → 755
> d) Application secrets (/opt/app/secrets/) → 700
> e) Shared team directory → 2775 (SGID)
> f) Cron script → 755

> **EXERCISE 3: Security Audit**
>
> a) Find all SUID binaries on the system
> b) Find all world-writable files
> c) Find files with no valid owner
> d) Check permissions on /etc/shadow (should be 640 or 000)
> e) List all users with UID 0 (should only be root)
> f) Check for users without passwords

> **DRILL: 10 Permission Challenges (2 min each)**
>
> 1. What does chmod 644 do? (rw-r--r--)
> 2. What does chmod 755 do? (rwxr-xr-x)
> 3. What does 2>&1 have to do with file descriptors?
> 4. Add execute permission for owner: chmod u+x
> 5. Change owner to deploy: chown deploy file
> 6. What is SUID? When is it needed?
> 7. What is the sticky bit? Where is it used?
> 8. What umask gives files 644 by default? (022)
> 9. How to prevent others from reading your SSH key?
> 10. Why never use chmod 777 in production?

---

## DAY 7: PROCESSES I
**Feb 21, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: process model, ps, top, signals
> 00:20-01:10 Hands-on: process monitoring and management
> 01:10-01:30 Drills: process challenges

### Understanding Processes

```
Every running program is a PROCESS.

Each process has:
  PID    — unique process ID
  PPID   — parent PID (who started it)
  UID    — user running it
  STATE  — running, sleeping, zombie, etc.
  Memory — RSS (physical), VSZ (virtual)
  CPU    — percentage of CPU time

Process lifecycle:
  Parent calls fork() → child process created
  Child calls exec() → loads new program
  Child calls exit() → becomes zombie
  Parent calls wait() → zombie cleaned up

KEY STATES:
  R = Running        (on CPU or in run queue)
  S = Sleeping       (waiting for event — normal)
  D = Disk sleep     (uninterruptible I/O — MANY = I/O problem!)
  Z = Zombie         (dead but parent hasn't called wait() — BUG!)
  T = Stopped        (paused by signal)
```

### ps — Process Snapshot

```bash
# Most useful forms:
ps aux                                       # all processes, BSD format
ps aux --sort=-%cpu | head -10               # top CPU consumers
ps aux --sort=-%mem | head -10               # top memory consumers
ps -eo pid,ppid,user,%cpu,%mem,stat,cmd --sort=-%cpu | head -20   # custom columns
ps -ef                                       # full format with PPID
ps -eo pid,ppid,stat,cmd | grep -w Z         # find zombies!

# Find specific processes:
ps aux | grep nginx                          # search by name
pgrep -la nginx                              # cleaner way
pgrep -c nginx                               # count processes

# Process tree:
pstree -p                                    # full tree with PIDs
pstree -p $(pgrep -f "my_app")              # tree for specific app
```

### top / htop — Live Monitoring

```bash
top
# KEY FIELDS IN top:
# us = user CPU    sy = system CPU    id = idle
# wa = I/O wait    st = steal (VM)    ni = nice
#
# INTERACTIVE KEYS:
# P = sort by CPU    M = sort by memory
# k = kill process   1 = show per-CPU
# c = full command   H = show threads
# q = quit

# htop (better alternative):
htop                                         # colored, mouse support
# F5 = tree view    F6 = sort by column
# F9 = kill          / = search
```

### Signals — Talking to Processes

```bash
# Signals are messages sent to processes.
# The kernel delivers them; processes can handle or ignore most.

# MOST IMPORTANT SIGNALS:
# SIGTERM (15) — "please stop" — DEFAULT for kill command, allows cleanup
# SIGKILL (9)  — "stop NOW" — cannot be caught/ignored, no cleanup
# SIGHUP  (1)  — "reload config" — used by nginx, apache, etc.
# SIGINT  (2)  — Ctrl+C — interrupt from keyboard
# SIGSTOP (19) — pause process (cannot be caught)
# SIGCONT (18) — resume paused process

# Send signals:
kill PID                         # sends SIGTERM (15) — graceful stop
kill -9 PID                      # sends SIGKILL — force kill (LAST RESORT!)
kill -HUP PID                    # sends SIGHUP — reload config
kill -STOP PID                   # pause process
kill -CONT PID                   # resume process
pkill -f "python my_script"      # kill by command name pattern
killall nginx                    # kill all processes with name "nginx"

# SRE WORKFLOW — always try graceful first:
# 1. kill PID                    (SIGTERM — give it 5-10 seconds)
# 2. still running? kill -9 PID  (SIGKILL — force, may lose data)

# Check if process exists:
kill -0 PID                      # signal 0 = just check, don't send anything
# Returns 0 if exists, non-zero if not

# Reload without restart:
kill -HUP $(cat /var/run/nginx.pid)     # nginx reloads config
systemctl reload nginx                   # same thing via systemd
```

### /proc/PID — Process Deep Dive

```bash
# Every process has a directory in /proc:
ls /proc/1/                              # PID 1 (systemd) details

cat /proc/1234/status                    # state, memory, threads
cat /proc/1234/cmdline | tr '\0' ' '     # full command line
cat /proc/1234/environ | tr '\0' '\n'    # environment variables
ls -l /proc/1234/fd/                     # open file descriptors
cat /proc/1234/limits                    # resource limits
readlink /proc/1234/cwd                  # current working directory
readlink /proc/1234/exe                  # executable path

# SRE: Find what a process has open:
ls -l /proc/1234/fd | wc -l             # count open FDs
ls -l /proc/1234/fd | grep socket       # count sockets
```

### DAY 7 EXERCISES

> **EXERCISE 1: Process Investigation**
>
> a) List all processes sorted by CPU usage
> b) Find the top 5 memory consumers
> c) Show the process tree for PID 1
> d) Find all processes owned by a specific user
> e) Find any zombie processes
> f) Count total processes running on the system

> **EXERCISE 2: Signal Practice**
>
> a) Start a background sleep: `sleep 300 &`
> b) Find its PID with `pgrep sleep`
> c) Send SIGSTOP to pause it (verify with ps)
> d) Send SIGCONT to resume it
> e) Send SIGTERM to stop it gracefully
> f) Start another, use SIGKILL — what's different?

> **EXERCISE 3: /proc Deep Dive**
>
> Pick any running process:
> a) Read its full command line from /proc/PID/cmdline
> b) Check its memory usage from /proc/PID/status
> c) Count its open file descriptors
> d) Check its resource limits
> e) Find its current working directory

> **DRILL: 10 Process Challenges (2 min each)**
>
> 1. List top 5 CPU-consuming processes
> 2. Find all nginx processes
> 3. Kill a process gracefully by PID
> 4. What signal does Ctrl+C send?
> 5. What is a zombie process?
> 6. Find the PPID of a process
> 7. Count open file descriptors for a PID
> 8. What does kill -0 do?
> 9. What's the difference between SIGTERM and SIGKILL?
> 10. Reload nginx without restarting it

---

## DAY 8: PROCESSES II — SERVICES & SCHEDULING
**Feb 22, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: systemd, journalctl, cron, background jobs
> 00:20-01:10 Hands-on: service management exercises
> 01:10-01:30 Drills: systemd challenges

### systemd — Service Management

```bash
# systemd is PID 1 — it manages ALL services.

# Service control:
systemctl status nginx                       # detailed status + recent logs
systemctl start nginx                        # start now
systemctl stop nginx                         # stop now
systemctl restart nginx                      # stop then start
systemctl reload nginx                       # reload config (no downtime)
systemctl enable nginx                       # start on boot
systemctl disable nginx                      # don't start on boot
systemctl is-active nginx                    # just "active" or "inactive"
systemctl is-enabled nginx                   # just "enabled" or "disabled"

# List services:
systemctl list-units --type=service          # all loaded services
systemctl list-units --type=service --state=running   # running only
systemctl list-units --type=service --state=failed    # FAILED (investigate!)

# Show service file:
systemctl cat nginx                          # see the unit file
systemctl show nginx                         # all properties
systemctl edit nginx --force                 # create override
```

### journalctl — Log Viewer

```bash
# journalctl is the interface to systemd's journal (logs).

journalctl -u nginx                          # logs for one service
journalctl -u nginx -f                       # follow (like tail -f)
journalctl -u nginx --since "1 hour ago"     # recent logs
journalctl -u nginx --since today            # today's logs
journalctl -p err                            # errors only (all services)
journalctl -p err --since today              # today's errors
journalctl -b                                # current boot only
journalctl -b -1                             # previous boot
journalctl --disk-usage                      # how much space logs use
journalctl -o json-pretty -u nginx | head -50  # JSON format
```

### cron — Scheduled Tasks

```bash
# cron runs commands on a schedule.

crontab -e                                   # edit YOUR crontab
crontab -l                                   # list YOUR crontab
sudo crontab -u deploy -e                    # edit another user's

# FORMAT:
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-7, 0 and 7 = Sunday)
# │ │ │ │ │
# * * * * * command

# EXAMPLES:
*/5 * * * * /opt/sre/health_check.sh >> /var/log/health.log 2>&1
#  Every 5 minutes, redirect both stdout and stderr to log

0 * * * * /opt/sre/collect_metrics.sh
#  Every hour at :00

0 2 * * * /opt/sre/cleanup.sh --days 30
#  Daily at 2:00 AM

0 9 * * 1 /opt/sre/weekly_report.sh
#  Every Monday at 9:00 AM

0 0 1 * * /opt/sre/monthly_report.sh
#  First day of every month at midnight

# COMMON MISTAKE: cron has a minimal PATH.
# Always use full paths in cron: /usr/bin/python3 instead of python3
```

### Background Jobs

```bash
# Run in background:
./long_script.sh &                           # run in background
nohup ./long_script.sh &                     # survives logout
nohup ./long_script.sh > output.log 2>&1 &   # background + log

# Job control:
jobs                                         # list background jobs
fg %1                                        # bring job 1 to foreground
bg %1                                        # resume stopped job in background
Ctrl+Z                                       # pause current process (SIGSTOP)

# Screen / tmux (persist sessions):
tmux new -s monitoring                       # create named session
tmux attach -t monitoring                    # reattach to session
# Ctrl+b d                                   # detach from session
tmux ls                                      # list sessions
```

### DAY 8 EXERCISES

> **EXERCISE 1: Service Management**
>
> a) List all running services
> b) Check status of sshd (or any installed service)
> c) Find any FAILED services — investigate why
> d) View the last 20 log lines for a service
> e) Follow a service's logs in real-time
> f) Show today's error-level logs across all services

> **EXERCISE 2: Cron Jobs**
>
> a) List your current crontab
> b) Write a cron entry for: every 10 minutes
> c) Write a cron entry for: daily at 3 AM
> d) Write a cron entry for: every Monday at 9 AM
> e) Write a cron entry for: first of month at midnight
> f) Add logging (redirect stdout and stderr to a log file)

> **EXERCISE 3: Background Jobs**
>
> a) Start a sleep 300 in the background
> b) List background jobs
> c) Bring it to foreground
> d) Pause it with Ctrl+Z
> e) Resume it in the background
> f) Practice with tmux: create, detach, reattach

> **DRILL: 10 Service & Scheduling Challenges (2 min each)**
>
> 1. Check if nginx is running with systemctl
> 2. View the last 50 journal lines for sshd
> 3. Find all failed services
> 4. What does `systemctl enable` do vs `start`?
> 5. Write a cron for every 5 minutes
> 6. Write a cron for daily at 2 AM
> 7. Run a command in background that survives logout
> 8. What does Ctrl+Z do to a process?
> 9. Reattach to a tmux session
> 10. Difference between restart and reload?

---

## DAY 9: I/O & REDIRECTION MASTERY
**Feb 24, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: tee, xargs, process substitution, here docs
> 00:20-01:10 Hands-on: advanced piping exercises
> 01:10-01:30 Drills: one-liner challenges

### tee — Write to File AND Screen

```bash
# tee copies stdin to BOTH stdout AND a file.
# Like a T-pipe: input splits into two directions.

command | tee output.log                     # display AND save
command | tee -a output.log                  # display AND append
command 2>&1 | tee output.log               # capture errors too

# SRE use: run a command and keep a record:
df -h | tee /tmp/disk_report.txt
ping -c 10 google.com | tee /tmp/ping_test.txt
```

### xargs — Build Commands from Input

```bash
# xargs takes stdin and passes it as ARGUMENTS to a command.

# Basic:
echo "file1 file2 file3" | xargs rm         # rm file1 file2 file3
find /tmp -name "*.tmp" | xargs rm           # delete found files
find /tmp -name "*.tmp" -print0 | xargs -0 rm  # handle spaces in names

# With placeholder:
cat servers.txt | xargs -I{} ping -c1 {}    # ping each server
cat servers.txt | xargs -I{} ssh {} uptime   # uptime on each server

# Parallel execution:
cat servers.txt | xargs -P4 -I{} ssh {} uptime   # 4 at a time!

# Line by line:
cat urls.txt | xargs -n1 curl -s -o /dev/null -w "%{url}: %{http_code}\n"

# SRE: Kill all matching processes:
pgrep -f "old_worker" | xargs kill
```

### Process Substitution

```bash
# <(command) treats command output as a file.
# Useful for commands that need FILE inputs, not stdin.

# Compare sorted output of two commands:
diff <(sort file1.txt) <(sort file2.txt)

# Compare configurations on two servers:
diff <(ssh web-01 cat /etc/nginx/nginx.conf) <(ssh web-02 cat /etc/nginx/nginx.conf)

# Combine two data sources:
paste <(cut -d: -f1 /etc/passwd) <(cut -d: -f3 /etc/passwd)
```

### Here Documents

```bash
# Feed multi-line input to a command:
cat << 'EOF'
Server: web-01
Status: running
CPU: 45%
EOF

# Useful in scripts for writing config files:
cat << 'EOF' > /etc/app/config.yaml
server:
  port: 8080
  host: 0.0.0.0
EOF
```

### DAY 9 EXERCISES

> **EXERCISE 1: tee Practice**
>
> a) Run `df -h` and save to file while also displaying
> b) Run `ps aux` and save + display, then grep the file for a process
> c) Append multiple commands' output to the same file with tee -a

> **EXERCISE 2: xargs Mastery**
>
> a) Delete all .tmp files in /tmp using find + xargs
> b) Ping 5 servers from a file: `cat servers.txt | xargs -I{} ping -c1 {}`
> c) Run uptime on multiple servers in parallel (simulate with localhost)
> d) Convert a list of filenames to a single-line comma-separated list

> **EXERCISE 3: SRE One-Liner Challenges**
>
> Build these one-liners:
> a) Top 10 IPs from access.log
> b) Percentage of 5xx errors in access.log
> c) Process with most open file descriptors
> d) Monitor disk usage every 5 seconds: `watch -n5 df -h`
> e) All listening ports with process names
> f) Count connections per IP per port
> g) Parse JSON from curl: `curl -s url | python3 -m json.tool`
> h) Compare config files on two servers (process substitution)
> i) Find all processes matching a pattern and kill them
> j) Ping 10 servers in parallel (xargs -P)

> **DRILL: 10 I/O Mastery Challenges (2 min each)**
>
> 1. Use tee to save and display a command's output
> 2. Use xargs with find to delete old files
> 3. Use xargs -P for parallel execution
> 4. Use diff with process substitution
> 5. Write a here document to create a config file
> 6. Redirect stderr and stdout to different files
> 7. Use /dev/null to silence a command
> 8. Count lines from a piped command
> 9. Use xargs -I{} for a placeholder
> 10. Chain 4 commands with pipes

---

## DAY 10: CAPSTONE LAB
**Feb 25-26, 2025 | 1.5 hrs each day**

### Scenario: Slow Application Investigation

> **Day 10 combines EVERYTHING from the past 2 weeks.**
>
> A web application is responding slowly. Users report 10+ second page loads.
> The normal response time is under 200ms. Walk through the investigation.

> **PHASE 1: System Overview (15 min)**
>
> Run these commands and record findings:
> ```bash
> uptime                          # load average
> free -h                         # memory
> df -h                           # disk space
> df -i                           # inode usage
> ss -s                           # connection summary
> dmesg -T | tail -20             # recent kernel messages
> systemctl list-units --state=failed   # failed services
> ```

> **PHASE 2: Process Analysis (15 min)**
>
> a) Find the application process (ps aux | grep app)
> b) Check its CPU and memory usage
> c) Count its open file descriptors: `ls /proc/PID/fd | wc -l`
> d) Check for zombie children: `ps -eo ppid,stat,cmd | grep PID`
> e) Is it writing to disk? `pidstat -d -p PID 1 5`

> **PHASE 3: Log Analysis (15 min)**
>
> a) Check application logs for errors: `grep ERROR /var/log/app.log | tail -20`
> b) Count errors per minute (frequency analysis)
> c) Look for patterns: timeouts, connection refused, out of memory
> d) Check web access log for slow URLs

> **PHASE 4: Root Cause (15 min)**
>
> Based on your findings, determine:
> a) Is it CPU-bound? (high CPU in top/ps)
> b) Is it memory-bound? (swap activity, OOM messages)
> c) Is it disk-bound? (high iowait, high disk util)
> d) Is it network-bound? (connection errors, DNS issues)
> e) Is it an application bug? (memory leak, connection leak)

> **PHASE 5: Document (10 min)**
>
> Write a brief incident report:
> - What was the symptom?
> - What commands did you run?
> - What was the root cause?
> - What was the fix?
> - How do you prevent this?

---

## WEEK 1-2 SUMMARY

> **FILE 01 COMPLETE — LINUX FUNDAMENTALS I**
>
> ✓ Filesystem: hierarchy, /proc, /etc, /var/log
> ✓ Navigation: cd, ls, find, locate, which
> ✓ File ops: cp, mv, rm, ln, tar, gzip, redirection
> ✓ Text processing: cat, head, tail, sort, uniq, cut, tr
> ✓ grep, sed, awk: pattern matching, substitution, column processing
> ✓ Users & permissions: chmod, chown, umask, SUID, SGID, sticky
> ✓ Processes: ps, top, kill, signals, /proc/PID, states
> ✓ Services: systemctl, journalctl, cron, tmux
> ✓ I/O: redirection, pipes, tee, xargs, process substitution
> ✓ 80+ exercises including full troubleshooting capstone
>
> **INTERVIEW PREP:**
> - "Walk through debugging a slow server" → USE method, top, free, df, ps, logs
> - "What does 2>&1 mean?" → Redirect stderr to wherever stdout points
> - "What is a zombie process?" → Finished but parent hasn't called wait()
> - "Explain file permissions 755" → rwxr-xr-x (owner all, group/other read+execute)
> - "How do you find what's using disk space?" → du -sh, find -size, lsof +L1
>
> Next: File 02 — Linux Fundamentals II (Weeks 3-4, Mar 1-12)
> Topics: Shell Scripting, Package Mgmt, Disk & Storage, SSH, Git
