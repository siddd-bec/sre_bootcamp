# LINUX & SYSTEMS — FILE 02 of 48
## LINUX FUNDAMENTALS II
### Shell Scripting, Package Management, Disk & Storage, SSH, Git
**Weeks 3-4 | Mar 1 - Mar 12, 2025 | 10 Days | 1.5 hrs/day**

---

> **BUILDING ON FILE 01**
>
> File 01: filesystem, commands, text processing, permissions, processes.
> File 02 adds AUTOMATION and INFRASTRUCTURE:
> - **Shell scripting** — automate everything (Days 1-4)
> - **Package management** — install and update software (Day 5)
> - **Disk & storage** — manage space, LVM, RAID (Day 6)
> - **SSH** — secure remote access (Day 7)
> - **Environment & Shell** — PATH, .bashrc, aliases (Day 8)
> - **Git** — version control (Day 9)
> - **Capstone** — automated provisioning script (Day 10)

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Bash Scripting I | shebang, variables, quoting, conditionals |
| 2 | Bash Scripting II | loops, functions, arrays, string ops |
| 3 | Bash Scripting III | arguments (getopts), error handling, debugging |
| 4 | Scripting Lab | Write 5 real SRE scripts |
| 5 | Package Management | apt, yum/dnf, repos, dependencies, pinning |
| 6 | Disk & Storage | partitions, mount, df, du, LVM, RAID |
| 7 | SSH & Remote Access | keys, config, tunnels, SCP, jump hosts |
| 8 | Environment & Shell Config | PATH, .bashrc, .profile, aliases |
| 9 | Git Essentials | init, add, commit, branch, merge, remote, PR |
| 10 | Capstone | Automated server provisioning script |

---

## DAYS 1-3: BASH SCRIPTING (Mar 1-3)

### Day 1: Script Structure & Variables

```bash
#!/bin/bash
# Shebang — tells OS to use bash interpreter. ALWAYS include.

# PRODUCTION SAFETY — add to EVERY script:
set -euo pipefail
# -e    Exit on ANY command failure
# -u    Error on undefined variables (catches typos!)
# -o pipefail  Pipe fails if ANY command in it fails
```

```bash
# === VARIABLES ===
# NO spaces around = sign!
NAME="Siddharth"                    # CORRECT
# NAME = "Siddharth"               # WRONG — bash thinks NAME is a command

# Quoting:
LITERAL='Hello $NAME'               # single quotes: literal → Hello $NAME
EXPANDED="Hello $NAME"              # double quotes: expanded → Hello Siddharth

# Command substitution:
HOSTNAME=$(hostname)                # capture command output
DATE=$(date +%Y-%m-%d)
KERNEL=$(uname -r)
DISK_PCT=$(df -h / | awk 'NR==2{print $5}')

# Arithmetic:
COUNT=$((5 + 3))                    # 8
PERCENT=$((USED * 100 / TOTAL))

# Special variables:
# $0    script name           $1, $2   positional arguments
# $#    number of arguments   $@       all arguments (as separate words)
# $?    exit code of last cmd $$       PID of current script
# $!    PID of last background process
```

### Day 1: Conditionals

```bash
# Always use [[ ]] (double brackets) in bash — safer than [ ]
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "Nginx found"
elif [[ -f /etc/apache2/apache2.conf ]]; then
    echo "Apache found"
else
    echo "No web server"
fi

# FILE TESTS:
# -f file exists (regular)    -d directory exists     -e anything exists
# -r readable                 -w writable             -x executable
# -s non-empty                -L symbolic link

# STRING TESTS:
# -z "$VAR"     empty string          -n "$VAR"    non-empty
# "$A" == "$B"  equal                 "$A" != "$B" not equal
# "$A" =~ ^[0-9]+$   regex match

# NUMERIC COMPARISONS:
# -eq equal  -ne not equal  -lt less than  -le less/equal  -gt greater  -ge greater/equal

# LOGICAL:
# &&  AND     ||  OR     !  NOT

# PRACTICAL SRE EXAMPLE:
CPU_USE=85
if [[ $CPU_USE -gt 90 ]]; then
    echo "CRITICAL: CPU at ${CPU_USE}%"
elif [[ $CPU_USE -gt 70 ]]; then
    echo "WARNING: CPU at ${CPU_USE}%"
else
    echo "OK: CPU at ${CPU_USE}%"
fi

# Check if command exists:
if command -v nginx &>/dev/null; then
    echo "nginx is installed"
else
    echo "nginx NOT found"
fi

# Inline conditional:
[[ -f /etc/nginx/nginx.conf ]] && echo "Found" || echo "Not found"
```

### Day 2: Loops

```bash
# FOR — iterate over list:
for server in web-01 web-02 db-01; do
    ping -c1 -W2 "$server" > /dev/null 2>&1 \
        && echo "✓ $server UP" || echo "✗ $server DOWN"
done

# FOR — iterate over files:
for logfile in /var/log/*.log; do
    size=$(du -sh "$logfile" | cut -f1)
    echo "$logfile: $size"
done

# FOR — C-style:
for ((i=1; i<=10; i++)); do
    echo "Iteration $i"
done

# WHILE READ — process file line by line (ESSENTIAL pattern):
while IFS= read -r line; do
    echo "Processing: $line"
done < servers.txt
# IFS= prevents whitespace trimming, -r prevents backslash escaping

# WHILE — monitoring loop:
while true; do
    CPU=$(top -bn1 | grep 'Cpu(s)' | awk '{print $2}')
    echo "$(date +%H:%M:%S) CPU: ${CPU}%"
    sleep 5
done

# LOOP CONTROL:
# break     exit loop entirely
# continue  skip to next iteration
```

### Day 2: Functions

```bash
# BASIC FUNCTION:
check_service() {
    local SVC=$1    # 'local' scopes variable to function
    if systemctl is-active --quiet "$SVC"; then
        echo "[OK] $SVC is running"
        return 0
    else
        echo "[FAIL] $SVC is stopped"
        return 1
    fi
}
check_service nginx
check_service postgresql

# FUNCTION WITH DEFAULTS:
check_port() {
    local HOST=$1
    local PORT=$2
    local TIMEOUT=${3:-5}    # default: 5 if not provided
    
    if nc -z -w"$TIMEOUT" "$HOST" "$PORT" 2>/dev/null; then
        echo "[OK] $HOST:$PORT open"
        return 0
    else
        echo "[FAIL] $HOST:$PORT closed"
        return 1
    fi
}
check_port "web-01" 80
check_port "db-01" 5432 10    # 10s timeout

# LOGGING FUNCTION — use in every production script:
log() {
    local LEVEL=$1; shift
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$LEVEL] $*" | tee -a "${LOG_FILE:-/dev/null}"
}
LOG_FILE="/var/log/my_script.log"
log INFO "Script started"
log WARN "Disk above 80%"
log ERROR "Service failed"
```

### Day 2: Arrays

```bash
# INDEXED ARRAY:
SERVERS=("web-01" "web-02" "web-03" "db-01")
echo "${SERVERS[0]}"          # first element
echo "${SERVERS[@]}"          # all elements
echo "${#SERVERS[@]}"         # count: 4
SERVERS+=("cache-01")         # append

for server in "${SERVERS[@]}"; do
    echo "Checking $server"
done

# ASSOCIATIVE ARRAY (like a dictionary):
declare -A PORTS
PORTS[web]=80
PORTS[db]=5432
PORTS[redis]=6379

for svc in "${!PORTS[@]}"; do
    echo "$svc → port ${PORTS[$svc]}"
done
```

### Day 2: Error Handling

```bash
# TRAP — run cleanup on exit or error:
cleanup() {
    rm -f /tmp/script_lock /tmp/script_temp_*
    log INFO "Cleanup complete"
}
trap cleanup EXIT                              # runs on ANY exit
trap 'echo "Error on line $LINENO"; cleanup' ERR  # runs on error

# RETRY FUNCTION:
retry() {
    local MAX=${1:-3}; local DELAY=${2:-5}; shift 2; local CMD="$@"
    for ((attempt=1; attempt<=MAX; attempt++)); do
        log INFO "Attempt $attempt/$MAX: $CMD"
        if eval "$CMD"; then return 0; fi
        [[ $attempt -lt $MAX ]] && sleep "$DELAY"
    done
    log ERROR "All $MAX attempts failed"
    return 1
}
retry 3 5 curl -sf http://api.example.com/health
```

### Day 3: Argument Parsing (getopts)

```bash
#!/bin/bash
set -euo pipefail

usage() {
    cat << EOF
Usage: $(basename "$0") [OPTIONS] HOSTNAME

Options:
    -v            Verbose output
    -e ENV        Environment (dev|staging|prod) [default: prod]
    -t TIMEOUT    Timeout in seconds [default: 5]
    -h            Show help

Examples:
    $(basename "$0") web-01
    $(basename "$0") -v -e staging -t 10 web-01
EOF
    exit 1
}

VERBOSE=false; ENVIRONMENT="prod"; TIMEOUT=5

while getopts "ve:t:h" opt; do
    case $opt in
        v) VERBOSE=true ;;
        e) ENVIRONMENT="$OPTARG" ;;
        t) TIMEOUT="$OPTARG" ;;
        h) usage ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))

[[ $# -eq 0 ]] && { echo "ERROR: HOSTNAME required"; usage; }
HOSTNAME=$1

# Validate:
case $ENVIRONMENT in
    dev|staging|prod) ;;
    *) echo "ERROR: Invalid environment: $ENVIRONMENT"; exit 1 ;;
esac

echo "Checking $HOSTNAME in $ENVIRONMENT (timeout: ${TIMEOUT}s)"
```

### Day 3: Debugging

```bash
# Debug mode — print each command before running:
bash -x script.sh                   # run with trace
set -x                              # enable inside script
set +x                              # disable

# Best practices checklist:
# ✅ set -euo pipefail
# ✅ [[ ]] not [ ]
# ✅ $(cmd) not `cmd`
# ✅ Quote all variables: "$VAR"
# ✅ local in functions
# ✅ trap cleanup EXIT
# ✅ Usage function
# ❌ Never: rm -rf $VAR/ (empty VAR → rm -rf /)
# ✅ Safe: rm -rf "${VAR:?ERROR}/"
```

### DAYS 1-3 EXERCISES

> **EXERCISE 1: Variable & Conditional Practice**
>
> Write a script that:
> a) Stores hostname, date, kernel, uptime in variables
> b) Checks disk usage, warns if > 80%
> c) Checks memory, warns if available < 500MB
> d) Checks load average, warns if > CPU count
> e) Prints overall status with color

> **EXERCISE 2: Loop & Function Library**
>
> a) Write `check_service()`, `check_port()`, `get_disk_usage()`, `log()`
> b) Read servers from a file, ping each in a loop
> c) Print ✓/✗ status and summary count
> d) Add retry logic with exponential backoff

> **EXERCISE 3: Production Script**
>
> Write a full script with:
> a) set -euo pipefail + trap cleanup EXIT
> b) getopts parsing (-v, -e ENV, -t TIMEOUT)
> c) Input validation (required args, valid values)
> d) Logging to file and console
> e) Meaningful exit codes (0=ok, 1=warning, 2=critical)

> **DRILL: 15 Scripting Challenges (2 min each)**
>
> 1. Assign command output to variable with $()
> 2. Test if a file exists with [[ -f ]]
> 3. Write a for loop over 5 items
> 4. Write a while read loop on a file
> 5. Write a function with local variables
> 6. Use ${VAR:-default} for default values
> 7. Parse -v and -e ENV with getopts
> 8. Use trap EXIT for cleanup
> 9. Add retry logic to a function
> 10. Create an array and loop over it
> 11. Check if a command exists
> 12. Use $? to check last exit code
> 13. Debug a script with bash -x
> 14. What does set -u prevent?
> 15. Why quote "$VAR" instead of $VAR?

---

## DAY 4: SCRIPTING LAB — 5 Real SRE Scripts
**Mar 4, 2025 | 1.5 hrs**

> Write each with: set -euo pipefail, logging, error handling, getopts.

> **SCRIPT 1: health-check.sh (20 min)**
> Check CPU (>80%=warn, >90%=crit), memory, disk per partition,
> key services running, key ports open. Exit 0/1/2.

> **SCRIPT 2: log-analyzer.sh (20 min)**
> Parse access.log: total requests, errors/hour, top 10 IPs,
> top 10 URLs, avg response time. Formatted report.

> **SCRIPT 3: backup.sh (20 min)**
> Takes source dir, backup dir, retention days.
> Creates tar.gz with date, verifies integrity, deletes old.
> Idempotent and logged.

> **SCRIPT 4: disk-alert.sh (15 min)**
> All partitions, alert >85%, top 10 files on full ones. Cron-ready.

> **SCRIPT 5: user-audit.sh (15 min)**
> Inactive 90-day users, no-password accounts, SUID binaries,
> world-writable files. Security report.

---

## DAY 5: PACKAGE MANAGEMENT
**Mar 5, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: apt, yum/dnf, repositories, pinning
> 00:20-01:10 Hands-on: install, update, audit packages
> 01:10-01:30 Drills: package challenges

### Debian/Ubuntu (apt)

```bash
# Update package lists (always first!):
sudo apt update

# Upgrade all packages:
sudo apt upgrade                           # safe upgrade
sudo apt full-upgrade                      # may add/remove packages

# Install / remove:
sudo apt install nginx                     # install
sudo apt install nginx=1.24.0-1           # specific version
sudo apt remove nginx                      # remove (keep config)
sudo apt purge nginx                       # remove everything
sudo apt autoremove                        # clean unused dependencies

# Search / info:
apt search nginx                           # find packages
apt show nginx                             # detailed info
apt list --installed                       # all installed packages
apt list --upgradable                      # pending updates

# Which package provides a file?
dpkg -S /usr/bin/curl                      # → curl: /usr/bin/curl
dpkg -L nginx                             # all files from nginx package

# Pin a version (prevent unwanted upgrades):
sudo apt-mark hold nginx                   # don't upgrade this
sudo apt-mark unhold nginx                 # allow upgrades again
```

### RHEL/CentOS (yum/dnf)

```bash
sudo yum update                            # or dnf update
sudo yum install httpd
sudo yum remove httpd
yum search nginx
rpm -qa | grep httpd                       # list installed
rpm -ql nginx                              # files in package
yum provides /usr/bin/curl                 # which package provides this
```

### SRE Package Patterns

```bash
# Check for security updates:
sudo apt list --upgradable 2>/dev/null | grep -i security

# Script: ensure required packages are installed:
REQUIRED=("curl" "jq" "net-tools" "htop" "tmux")
for pkg in "${REQUIRED[@]}"; do
    if ! dpkg -l "$pkg" &>/dev/null; then
        echo "Installing $pkg..."
        sudo apt install -y "$pkg"
    else
        echo "✓ $pkg already installed"
    fi
done
```

### DAY 5 EXERCISES

> **EXERCISE 1:** Count installed packages. Find which package provides `ss`. Install htop, verify.
> **EXERCISE 2:** List pending security updates. Pin a package version. Write a script to ensure 5 required packages are installed.
> **EXERCISE 3:** Compare package versions between two servers (simulate with lists).

---

## DAY 6: DISK & STORAGE
**Mar 6, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: partitions, filesystems, LVM, RAID
> 00:20-01:10 Hands-on: disk management exercises
> 01:10-01:30 Drills: storage challenges

### Disk Commands

```bash
# Space usage:
df -h                           # filesystem usage (human-readable)
df -i                           # INODE usage (hidden issue!)
du -sh /var/log/*               # directory sizes
du -sh /var/log/* | sort -rh | head -10   # largest dirs

# Block devices:
lsblk                           # tree of all block devices
fdisk -l                        # partition tables
blkid                           # UUIDs and filesystem types

# Mount:
mount /dev/sdb1 /mnt            # mount a partition
umount /mnt                     # unmount
cat /etc/fstab                  # permanent mounts (survive reboot)

# Mount line in fstab:
# UUID=xxxx-xxxx  /data  ext4  defaults  0  2
```

### LVM (Logical Volume Manager)

```
LVM adds a flexible layer between disks and filesystems.
Instead of fixed partitions, you can resize on the fly.

Physical Volume (PV)  →  Volume Group (VG)  →  Logical Volume (LV)
   /dev/sda1               vg_data              lv_logs
   /dev/sdb1                                     lv_data
```

```bash
# Create:
pvcreate /dev/sdb1                       # make physical volume
vgcreate vg_data /dev/sdb1               # create volume group
lvcreate -L 50G -n lv_logs vg_data       # create logical volume

# Format and mount:
mkfs.ext4 /dev/vg_data/lv_logs
mount /dev/vg_data/lv_logs /var/log

# EXTEND (the SRE power move — no downtime!):
lvextend -L +10G /dev/vg_data/lv_logs    # add 10GB
resize2fs /dev/vg_data/lv_logs           # grow filesystem

# View:
pvs                                       # physical volumes
vgs                                       # volume groups
lvs                                       # logical volumes
```

### Disk Full Emergency — SRE Response

```bash
# 1. Find what's using space:
du -sh /* 2>/dev/null | sort -rh | head -10

# 2. Find large files:
find / -type f -size +1G 2>/dev/null

# 3. HIDDEN GOTCHA — deleted files still open:
lsof +L1
# Shows files that have been deleted BUT are still open by a process.
# The space won't be freed until the process closes the file!
# Fix: restart the process, or truncate: > /proc/PID/fd/FD_NUMBER

# 4. Truncate (don't delete!) large log:
> /var/log/huge.log              # empties without breaking log rotation
# NOT: rm /var/log/huge.log     # process still holds the file!

# 5. INODE EXHAUSTION (Interview favorite!):
df -i
# Shows 100% inodes used? Can't create files even with free space!
# Cause: millions of tiny files (mail queue, session files, cache)
# Fix: find and delete the tiny files
find /tmp -type f -name "sess_*" -delete
```

### DAY 6 EXERCISES

> **EXERCISE 1:** Run df -h and df -i. Which partitions are most used? Any inode concerns?
> **EXERCISE 2:** Find the top 20 largest files on the system. Find the top 10 largest directories.
> **EXERCISE 3:** Explain: "df shows 50% free but app can't write files." → Check df -i (inodes).
> **EXERCISE 4:** Explain LVM: what is PV, VG, LV? How to extend a volume with no downtime?
> **EXERCISE 5:** Practice lsof +L1 — what does it show and why does it matter?

---

## DAY 7: SSH & REMOTE ACCESS
**Mar 7, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: SSH keys, config, tunnels, security
> 00:20-01:10 Hands-on: SSH configuration exercises
> 01:10-01:30 Drills: SSH challenges

### SSH Key Authentication

```bash
# Generate key pair (use ed25519 — modern, secure):
ssh-keygen -t ed25519 -C "siddharth@visa"
# Creates: ~/.ssh/id_ed25519 (private) + ~/.ssh/id_ed25519.pub (public)

# PERMISSIONS (must be correct or SSH refuses!):
chmod 700 ~/.ssh                   # directory
chmod 600 ~/.ssh/id_ed25519        # private key (OWNER ONLY)
chmod 644 ~/.ssh/id_ed25519.pub    # public key
chmod 600 ~/.ssh/authorized_keys   # on remote server
chmod 644 ~/.ssh/config            # SSH config file

# Copy public key to remote server:
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
# Or manually: append public key to remote ~/.ssh/authorized_keys

# Connect:
ssh user@server                    # uses default key
ssh -i ~/.ssh/special_key user@server   # specific key
```

### SSH Config (~/.ssh/config)

```bash
# Saves you from typing long commands every time!

# ~/.ssh/config:
Host web-prod
    HostName 10.0.1.50
    User deploy
    IdentityFile ~/.ssh/deploy_key
    Port 22

Host db-prod
    HostName 10.0.2.10
    User admin
    IdentityFile ~/.ssh/db_key

Host *.staging
    User deploy
    ProxyJump bastion-staging
    IdentityFile ~/.ssh/staging_key

Host bastion-staging
    HostName bastion.staging.company.com
    User siddharth

# Now you can just type:
ssh web-prod                       # instead of long ssh command
scp file.txt web-prod:/tmp/        # uses config automatically
```

### SSH Tunneling

```bash
# LOCAL TUNNEL — access remote service through local port:
ssh -L 8080:localhost:3000 web-prod
# Now localhost:8080 → web-prod:3000
# Use for: accessing internal dashboards, databases

# LOCAL TUNNEL — access remote database:
ssh -L 5433:db-internal:5432 bastion
# Now localhost:5433 → db-internal:5432 (through bastion)

# JUMP HOST (ProxyJump) — SSH through a bastion:
ssh -J bastion user@internal-server
# Or in config: ProxyJump bastion

# SCP — copy files:
scp file.txt web-prod:/tmp/                        # upload
scp web-prod:/var/log/app.log ./                   # download
scp -r web-prod:/etc/nginx/ ./nginx-backup/        # directory

# rsync — better than scp (incremental, resumable):
rsync -avz /local/dir/ user@server:/remote/dir/
# -a archive  -v verbose  -z compress
# Only transfers CHANGED files — much faster for repeated syncs
```

### SSH Security Hardening

```bash
# /etc/ssh/sshd_config (server side):
PermitRootLogin no                 # never allow root SSH
PasswordAuthentication no          # keys only
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers deploy siddharth        # whitelist users
Protocol 2                         # SSH v2 only

# After changes:
sudo systemctl restart sshd
# WARNING: test from a SECOND terminal before closing current session!
```

### DAY 7 EXERCISES

> **EXERCISE 1:** Generate an ed25519 key pair. Verify correct permissions (700, 600, 644).
> **EXERCISE 2:** Create SSH config with 3 host entries. Use aliases to connect.
> **EXERCISE 3:** Set up a local tunnel. Explain: how does `-L 8080:localhost:3000` work?
> **EXERCISE 4:** Practice scp and rsync file transfers.
> **EXERCISE 5:** List 5 SSH hardening steps for a production server.

---

## DAY 8: ENVIRONMENT & SHELL CONFIG
**Mar 8, 2025 | 1.5 hrs**

### Shell Initialization Files

```bash
# LOGIN SHELL (ssh, login):      /etc/profile → ~/.bash_profile → ~/.bashrc
# NON-LOGIN SHELL (new terminal): ~/.bashrc only

# RULE: Put everything in ~/.bashrc (sourced by both types)

# ~/.bashrc additions for SRE:
# History:
export HISTSIZE=50000                       # commands in memory
export HISTFILESIZE=100000                  # commands on disk
export HISTTIMEFORMAT="%F %T "             # timestamps!
export HISTCONTROL=ignoredups:erasedups     # no duplicates

# PATH:
export PATH="$HOME/.local/bin:$HOME/scripts:$PATH"

# Aliases:
alias ll='ls -lah'
alias la='ls -la'
alias grep='grep --color=auto'
alias k='kubectl'
alias tf='terraform'
alias g='git'
alias gs='git status'
alias gd='git diff'
alias gc='git commit'
alias gp='git push'
alias watch='watch -n2'
alias ports='ss -tlnp'
alias psg='ps aux | grep -v grep | grep'

# SRE Functions:
sshcheck() { ssh -o ConnectTimeout=3 -o BatchMode=yes "$1" echo ok 2>/dev/null && echo "✓ $1" || echo "✗ $1"; }

# Prompt (shows hostname and current directory):
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
```

### Environment Variables

```bash
# View:
env                                # all environment variables
echo $PATH                        # specific variable
printenv HOME                     # another way

# Set for current session:
export MY_VAR="value"              # available to child processes
MY_VAR="value"                     # only in current shell

# Set permanently:
echo 'export MY_VAR="value"' >> ~/.bashrc
source ~/.bashrc                   # reload without logout
```

### DAY 8 EXERCISES

> **EXERCISE 1:** Configure .bashrc with: history settings, 5 aliases, custom prompt, PATH addition.
> **EXERCISE 2:** Create 3 SRE aliases and 2 shell functions. Source and test.
> **EXERCISE 3:** Explain: what's the difference between `export VAR=x` and `VAR=x`?

---

## DAY 9: GIT ESSENTIALS
**Mar 9-10, 2025 | 1.5 hrs each**

> 00:00-00:20 Teach: Git concepts, workflow
> 00:20-01:10 Hands-on: full Git workflow
> 01:10-01:30 Drills: Git challenges

### Core Git Workflow

```bash
# === SETUP ===
git config --global user.name "Siddharth"
git config --global user.email "siddharth@example.com"
git config --global init.defaultBranch main

# === INITIALIZE ===
mkdir my-project && cd my-project
git init                             # creates .git directory

# === BASIC WORKFLOW ===
# 1. Make changes
echo "# My SRE Project" > README.md

# 2. Stage changes (choose what to commit):
git add README.md                    # stage specific file
git add .                            # stage everything

# 3. Commit (save snapshot):
git commit -m "Initial commit: add README"

# 4. View history:
git log                              # full log
git log --oneline                    # compact (one line per commit)
git log --oneline --graph            # visual branch graph

# === STATUS & DIFF ===
git status                           # what's changed?
git diff                             # unstaged changes
git diff --staged                    # staged changes (about to commit)
```

### Branching and Merging

```bash
# === BRANCHES ===
# Branches let you work on features without affecting main code.

git branch                           # list branches
git branch feature/monitoring        # create branch
git checkout feature/monitoring      # switch to branch
git checkout -b feature/alerting     # create + switch (shorthand)

# Make changes on branch:
echo "alert config" > alerting.yaml
git add alerting.yaml
git commit -m "feat: add alerting config"

# === MERGE ===
git checkout main                    # switch back to main
git merge feature/alerting           # merge feature into main

# If conflict:
# 1. Git marks conflicts in files with <<<<<<< ======= >>>>>>>
# 2. Edit files to resolve
# 3. git add resolved_file
# 4. git commit

# === DELETE BRANCH ===
git branch -d feature/alerting       # delete merged branch

# === REMOTE ===
git remote add origin https://github.com/user/repo.git
git push -u origin main              # push to remote (first time)
git push                             # subsequent pushes
git pull                             # fetch + merge from remote
git clone https://github.com/user/repo.git   # download repo
```

### SRE Git Patterns

```bash
# Feature branch workflow:
git checkout -b feature/add-prometheus-config
# ... make changes ...
git add .
git commit -m "feat: add Prometheus scrape config for web tier"
git push -u origin feature/add-prometheus-config
# Create Pull Request on GitHub/GitLab for review
# After review: merge PR, delete branch

# Good commit messages (conventional commits):
git commit -m "fix: resolve connection timeout in health checker"
git commit -m "feat: add disk usage monitoring to dashboard"
git commit -m "docs: update runbook for database failover"
git commit -m "chore: upgrade prometheus-client to 0.19.0"

# Undo mistakes:
git checkout -- file.txt             # discard unstaged changes
git reset HEAD file.txt              # unstage a file
git revert HEAD                      # create new commit that undoes last
git stash                            # save changes for later
git stash pop                        # restore stashed changes

# .gitignore:
cat > .gitignore << 'EOF'
*.pyc
__pycache__/
.env
.venv/
*.log
node_modules/
.terraform/
*.tfstate
EOF
```

### DAY 9 EXERCISES

> **EXERCISE 1: Full Git Workflow**
>
> a) Init a repo, create README, make initial commit
> b) Create a branch, make 3 commits on it
> c) Switch back to main, make 1 commit
> d) Merge branch into main
> e) View log with --oneline --graph

> **EXERCISE 2: Conflict Resolution**
>
> a) Create two branches from same point
> b) Edit the same line in the same file on both
> c) Merge one into main (succeeds)
> d) Merge the other (conflict!)
> e) Resolve the conflict, commit

> **EXERCISE 3: SRE Git Setup**
>
> a) Create a proper .gitignore for Python/SRE projects
> b) Practice git stash (save, list, pop)
> c) Practice git revert (undo a commit safely)
> d) Write 5 good commit messages following conventional commits

> **DRILL: 10 Git Challenges (2 min each)**
>
> 1. Initialize a repo and make first commit
> 2. Create and switch to a new branch
> 3. View commit history (compact)
> 4. Stage only one specific file
> 5. Show what's changed but not staged
> 6. Merge a branch into main
> 7. Stash changes and pop them back
> 8. Undo the last commit (git revert HEAD)
> 9. Write a .gitignore for Python projects
> 10. What's the difference between git pull and git fetch?

---

## DAY 10: CAPSTONE — SERVER PROVISIONING SCRIPT
**Mar 11-12, 2025 | 1.5 hrs each**

> **Build `provision.sh` — a complete server provisioning script.**
>
> This combines EVERYTHING from Weeks 3-4.

> **Requirements:**
> a) Accepts: hostname, environment (dev/staging/prod), role (web/db/app)
> b) Uses set -euo pipefail, trap cleanup EXIT
> c) Argument parsing with getopts + validation
> d) Logging with timestamps to file
> e) Updates system packages (apt update && upgrade)
> f) Installs base tools: curl, jq, htop, tmux, net-tools
> g) Creates deploy user with SSH key
> h) Hardens SSH (no root login, no password auth)
> i) Configures basic firewall (ufw: allow SSH, HTTP, HTTPS)
> j) Installs role-specific packages:
>    - web: nginx
>    - db: postgresql
>    - app: python3, python3-pip
> k) Sets up node_exporter for Prometheus monitoring
> l) Writes completion report (JSON)
> m) IDEMPOTENT — safe to run multiple times
> n) Supports --dry-run mode

---

## WEEKS 3-4 SUMMARY

> **FILE 02 COMPLETE — LINUX FUNDAMENTALS II**
>
> ✓ Bash scripting: variables, conditionals, loops, functions, arrays
> ✓ Error handling: set -euo pipefail, trap, retry
> ✓ Argument parsing: getopts, validation, usage
> ✓ 5 real SRE scripts: health check, log analyzer, backup, disk alert, audit
> ✓ Package management: apt, yum/dnf, pinning, security updates
> ✓ Disk & storage: df, du, lsof +L1, LVM, inode exhaustion
> ✓ SSH: keys, config, tunnels, jump hosts, hardening
> ✓ Environment: .bashrc, PATH, aliases, functions
> ✓ Git: init, branch, merge, conflict, .gitignore, workflows
> ✓ 60+ exercises including server provisioning capstone
>
> **INTERVIEW PREP:**
> - "What does set -euo pipefail do?" → -e exits on error, -u errors on
>   undefined variables, -o pipefail fails pipe if any command fails
> - "How do you make a script idempotent?" → Check before acting
> - "Disk full but df shows space?" → Check df -i (inodes) or lsof +L1
> - "How do you extend a volume with no downtime?" → lvextend + resize2fs
> - "SSH best practices?" → ed25519 keys, no root login, no password auth
>
> Next: File 03 — Linux Networking (Weeks 5-6, Mar 15-26)
