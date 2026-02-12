# LINUX & SYSTEMS — FILE 02 of 48
## LINUX FUNDAMENTALS II
### Shell Scripting, Package Mgmt, Disk & Storage, SSH, Git
**Weeks 3-4 | Mar 1 - Mar 12, 2025 | 10 Days | 1.5 hrs/day**

---

> **BUILDING ON FILE 01**
> File 01: filesystem, commands, text processing, permissions, processes.
> File 02 adds: scripting (automation!), packages, storage, SSH, Git.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Bash Scripting I | shebang, variables, quoting, conditionals |
| 2 | Bash Scripting II | loops, functions, arrays, string ops |
| 3 | Bash Scripting III | input, arguments, error handling, debugging |
| 4 | Scripting Lab | Write 5 real SRE scripts |
| 5 | Package Management | apt, yum, repos, dependencies, security updates |
| 6 | Disk & Storage | partitions, mount, df, du, LVM, RAID |
| 7 | SSH & Remote Access | keys, config, tunnels, SCP, jump hosts |
| 8 | Environment & Shell Config | PATH, .bashrc, .profile, aliases |
| 9 | Git Essentials | init, add, commit, branch, merge, remote, PR |
| 10 | Capstone | Automated server provisioning script |

---

## DAYS 1-3: BASH SCRIPTING
**Mar 1-3, 2025**

### Script Structure & Variables

```bash
#!/bin/bash
set -euo pipefail  # ALWAYS in production scripts
# -e: exit on error  -u: error on undefined  -o pipefail: pipe fails properly

# Variables (NO spaces around =):
NAME='Siddharth'           # single quotes: literal
GREETING="Hello $NAME"     # double quotes: expansion
HOSTNAME=$(hostname)        # command substitution
DATE=$(date +%Y-%m-%d)

# Special variables:
# $0=script name  $1,$2=args  $#=arg count  $?=exit code  $$=PID  $@=all args
```

### Conditionals

```bash
if [[ -f /etc/nginx/nginx.conf ]]; then
    echo "Nginx found"
elif [[ -f /etc/apache2/apache2.conf ]]; then
    echo "Apache found"
else
    echo "No web server"
fi

# File tests: -f file, -d dir, -r readable, -w writable, -x executable, -s non-empty
# String: -z empty, -n not-empty, == !=
# Numeric: -eq -ne -lt -le -gt -ge
# Logical: && (and), || (or), ! (not)
```

### Loops

```bash
# for loop:
for server in web-01 web-02 db-01; do
    ping -c1 -W2 $server >/dev/null 2>&1 && echo "$server: UP" || echo "$server: DOWN"
done

# C-style:
for ((i=1; i<=10; i++)); do echo "Iteration $i"; done

# while read (process file line by line — VERY useful):
while IFS= read -r line; do
    echo "Processing: $line"
done < servers.txt

# Infinite with break:
while true; do
    CPU=$(top -bn1 | grep 'Cpu(s)' | awk '{print $2}')
    echo "CPU: $CPU%"
    sleep 5
done
```

### Functions & Error Handling

```bash
# Functions:
check_service() {
    local SVC=$1    # local variable
    systemctl is-active --quiet $SVC && echo "[OK] $SVC" || echo "[FAIL] $SVC"
}
check_service nginx
check_service postgresql

# Error handling:
trap 'echo "Error on line $LINENO"; cleanup' ERR
trap cleanup EXIT   # runs on exit (success or failure)

# Argument parsing:
usage() { echo "Usage: $0 [-v] [-e env] host"; exit 1; }
VERBOSE=false; ENV='prod'
while getopts 've:h' opt; do
    case $opt in
        v) VERBOSE=true ;;
        e) ENV=$OPTARG ;;
        *) usage ;;
    esac
done
shift $((OPTIND - 1))
[ $# -eq 0 ] && usage
SERVER=$1

# Logging:
log() { echo "$(date +%F_%T) [$1] ${@:2}" | tee -a /var/log/script.log; }
log INFO "Deployment starting"
```

### SCRIPTING EXERCISES (24 total)

**Variables & Conditionals (1-4):** Store hostname/date/uptime, check service status, check disk/memory/load alerts, validate arguments.
**Loops (5-8):** Ping servers from file, monitor CPU loop, process .log files, retry with backoff.
**Functions (9-12):** get_disk_usage/get_memory/get_load, check_port with nc, log() with levels, generic retry().
**Production (13-16):** Add set -euo pipefail, add trap cleanup, parse with getopts, make idempotent.

---

## DAY 4: SCRIPTING LAB
**Mar 4, 2025 — Write 5 Real SRE Scripts**

**Script 1: health-check.sh** — Check services, ports, disk, memory. Exit 1 if critical.
**Script 2: log-analyzer.sh** — Parse access.log: errors/hour, top IPs, top URLs, avg response time.
**Script 3: backup.sh** — Tar.gz with date, keep last 7, verify integrity, log actions.
**Script 4: disk-alert.sh** — All partitions, alert >85%, show top files. Cron-ready.
**Script 5: user-audit.sh** — Inactive users, no-password, SUID bins, world-writable. Security report.

---

## DAY 5: PACKAGE MANAGEMENT
**Mar 5, 2025**

```bash
# Debian/Ubuntu:
apt update && apt upgrade           # refresh + upgrade
apt install nginx                   # install
apt remove --purge nginx            # remove completely
dpkg -l | grep nginx                # list installed
dpkg -L nginx                       # files from package
apt list --upgradable               # pending updates

# RHEL/CentOS:
yum update / dnf update
yum install httpd
rpm -qa | grep httpd
```

**Exercises:** Count packages, find which package provides a binary, install/verify nginx, pin version, script to check/install required packages.

---

## DAY 6: DISK & STORAGE
**Mar 6, 2025**

```bash
df -h               # filesystem usage
df -i               # inode usage (hidden issue!)
du -sh /var/log/*   # directory sizes
lsblk               # block devices
mount /dev/sdb1 /mnt

# LVM: PV -> VG -> LV
lvextend -L +10G /dev/vg0/data && resize2fs /dev/vg0/data

# Disk full emergency:
# 1. du -sh /* | sort -rh | head
# 2. find / -size +1G
# 3. lsof +L1 (deleted files still open = space not freed!)
# 4. truncate > /var/log/huge.log
```

> **INODE EXHAUSTION**
> df -h shows space but df -i shows 100% = cannot create files.
> Millions of tiny files exhaust inodes. Interview: "Disk shows free space but apps can't write."
> Answer: "Check df -i. Likely inode exhaustion."

---

## DAY 7: SSH & REMOTE ACCESS
**Mar 7, 2025**

```bash
# Generate key:
ssh-keygen -t ed25519 -C 'siddharth@visa'

# SSH config (~/.ssh/config):
Host web-prod
    HostName 10.0.1.50
    User deploy
    IdentityFile ~/.ssh/deploy_key
Host *.staging
    ProxyJump bastion-staging

# Tunneling:
ssh -L 8080:localhost:3000 web-prod   # local forward
ssh -J bastion user@internal          # jump host

# File transfer:
scp file.txt user@server:/tmp/
rsync -avz /local/ user@server:/remote/
```

**Exercises:** Generate keys (600/644 perms), SSH config for 3 hosts, local tunnel, harden sshd, script SSH to 5 servers.

---

## DAYS 8-9: ENVIRONMENT & GIT
**Mar 8-10, 2025**

```bash
# Environment:
export PATH="$HOME/.local/bin:$PATH"
# .bashrc: aliases, PATH, HISTSIZE=50000, custom prompt

# Git workflow:
git checkout -b feature/monitoring
git add . && git commit -m 'add prometheus config'
git push origin feature/monitoring
# Create Pull Request for code review
```

**Exercises:** Configure .bashrc, Git init/branch/5 commits/merge, resolve conflict, write .gitignore, git stash/revert.

---

## DAY 10: CAPSTONE — SERVER PROVISIONING SCRIPT
**Mar 11, 2025**

Write `provision.sh` that:
- Accepts: hostname, environment, role (web/db/app)
- Uses set -euo pipefail + trap cleanup
- Updates packages, installs base tools
- Creates deploy user with SSH key
- Hardens SSH, configures firewall (ufw)
- Installs role-specific packages
- Sets up node_exporter for Prometheus
- Writes completion report. Idempotent (safe to run twice).

---

> **FILE 02 COMPLETE — LINUX FUNDAMENTALS II**
> Next: File 03 — Linux Networking (Weeks 5-6, Mar 15-26)
