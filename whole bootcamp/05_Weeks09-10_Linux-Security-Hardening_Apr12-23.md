# LINUX & SYSTEMS — FILE 05 of 48
## LINUX SECURITY & HARDENING
### SELinux/AppArmor, SSH Hardening, Auditing, PAM, TLS/Certificates, Container Security
**Weeks 9-10 | Apr 12 - Apr 23, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY SECURITY IS NON-NEGOTIABLE FOR SRE**
>
> Apple builds products trusted by billions. Security isn't a feature — it's the foundation.
> Every SRE interview includes: "How do you harden a Linux server?"
> Every incident postmortem asks: "Could this have been prevented with better security?"
>
> After these 2 weeks you will:
> - Harden a Linux server from a fresh install to production-ready
> - Understand and configure SELinux/AppArmor mandatory access controls
> - Set up comprehensive auditing and intrusion detection
> - Manage TLS certificates and understand the PKI chain
> - Secure containers and understand the attack surface
> - Answer any interview question about Linux security confidently

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | Security Fundamentals | Threat models, defense in depth, least privilege |
| 2 | SSH Hardening | Key auth, config hardening, 2FA, fail2ban |
| 3 | Firewall Deep Dive | iptables advanced, nftables, zones, rate limiting |
| 4 | SELinux / AppArmor | Mandatory access control, policies, troubleshooting |
| 5 | Practice Lab I | Harden a server from scratch |
| 6 | Auditing & Logging | auditd, rsyslog, log integrity, SIEM basics |
| 7 | PAM & Authentication | PAM modules, password policy, MFA, LDAP |
| 8 | TLS & Certificates | PKI, openssl, Let's Encrypt, cert management |
| 9 | Container Security | Image scanning, rootless, seccomp, capabilities |
| 10 | Capstone | Security audit + hardening of production server |

---

## DAY 1: SECURITY FUNDAMENTALS
**Apr 12, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: Threat models, defense in depth, principle of least privilege
> 00:20-01:10 Hands-on: Security assessment of a default install
> 01:10-01:30 Drills: Identify vulnerabilities

### Core Security Principles

```
DEFENSE IN DEPTH — Multiple layers, no single point of failure:
  Layer 1: Network (firewalls, VPN, network segmentation)
  Layer 2: Host (OS hardening, patching, access control)
  Layer 3: Application (input validation, auth, encryption)
  Layer 4: Data (encryption at rest, backups, access logging)

LEAST PRIVILEGE — Every user/process gets minimum permissions needed:
  - Don't run services as root
  - Don't give sudo to everyone
  - Don't use 777 permissions
  - Don't store secrets in plaintext

ZERO TRUST — Never trust, always verify:
  - Authenticate every request
  - Encrypt all traffic (even internal)
  - Log everything
  - Assume breach
```

### Security Assessment Checklist

```bash
# QUICK SECURITY AUDIT (run on any server):

# 1. Who can log in?
cat /etc/passwd | awk -F: '$7 != "/usr/sbin/nologin" && $7 != "/bin/false"'
cat /etc/shadow | awk -F: '$2 == "" || $2 == "!" {print $1}'  # no password

# 2. Who has sudo?
grep -E '^%sudo|^%wheel|^%admin' /etc/sudoers
cat /etc/sudoers.d/*

# 3. SSH configuration:
grep -E 'PermitRootLogin|PasswordAuthentication|PubkeyAuthentication' /etc/ssh/sshd_config

# 4. Listening services (attack surface):
ss -tlnp    # what is exposed?

# 5. SUID/SGID binaries (privilege escalation risk):
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# 6. World-writable files:
find / -perm -o+w -type f 2>/dev/null | grep -v '/proc\|/sys\|/dev'

# 7. Unowned files:
find / -nouser -o -nogroup 2>/dev/null

# 8. Installed packages with known CVEs:
apt list --installed 2>/dev/null | wc -l
# Check: https://security-tracker.debian.org or use tools like Trivy

# 9. Kernel and OS version:
uname -r
cat /etc/os-release

# 10. Running as root unnecessarily:
ps aux | awk '$1 == "root"' | grep -v '\[.*\]' | wc -l
```

### DAY 1 EXERCISES

1. Run the full security audit checklist on your system. Document findings.
2. List all users who can log in (have a valid shell). Should all of them?
3. Check SSH config: is root login allowed? Password auth enabled?
4. List all listening ports. For each: is this service needed? Who can reach it?
5. Find all SUID binaries. Research: which are normal, which are suspicious?
6. Check sudo configuration. Who has unlimited sudo? Should they?
7. Find world-writable files outside /tmp. Why are they a risk?
8. Create a "security baseline" document for your system.

---

## DAY 2: SSH HARDENING
**Apr 13, 2025 | 1.5 hrs**

### Hardening /etc/ssh/sshd_config

```bash
# CRITICAL SETTINGS (edit /etc/ssh/sshd_config):

# Disable root login:
PermitRootLogin no

# Disable password authentication (keys only):
PasswordAuthentication no
PubkeyAuthentication yes

# Restrict to specific users/groups:
AllowUsers deploy siddharth
# or: AllowGroups sre-team

# Use strong key exchange and ciphers:
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com

# Timeout settings:
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30

# Disable unnecessary features:
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no   # unless needed for tunnels

# Limit authentication attempts:
MaxAuthTries 3
MaxSessions 3

# Use non-standard port (security through obscurity — minor benefit):
# Port 2222

# After changes:
sshd -t                 # test config syntax
systemctl restart sshd  # apply (keep existing session open!)
```

### fail2ban — Brute Force Protection

```bash
# Install:
apt install fail2ban

# Config: /etc/fail2ban/jail.local
# [sshd]
# enabled = true
# port = ssh
# filter = sshd
# logpath = /var/log/auth.log
# maxretry = 3
# bantime = 3600        # 1 hour ban
# findtime = 600        # within 10 minutes

# Commands:
fail2ban-client status sshd          # check banned IPs
fail2ban-client set sshd unbanip IP  # unban specific IP
```

> **SSH KEY BEST PRACTICES**
>
> - Use ed25519 keys (faster, more secure than RSA)
> - Always set a passphrase on private keys
> - Use ssh-agent so you don't retype passphrase
> - Rotate keys annually
> - Never share private keys
> - Permission: private key 600, public key 644, .ssh dir 700
> - authorized_keys file: 600

### DAY 2 EXERCISES

1. Harden sshd_config with all settings above. Test with `sshd -t`.
2. Disable password auth. Verify you can still log in with key.
3. Set up fail2ban for SSH. Test: trigger a ban, verify, unban.
4. Create separate SSH keys for different purposes (work, personal).
5. Set up SSH certificate-based auth (advanced: ssh-keygen -s CA_key).
6. Audit /var/log/auth.log: any failed login attempts? From where?
7. Configure AllowUsers to restrict SSH to specific accounts only.
8. Test: what happens if you lock yourself out? Have a recovery plan.

---

## DAY 3: FIREWALL DEEP DIVE
**Apr 14, 2025 | 1.5 hrs**

### Advanced iptables

```bash
# Connection tracking (stateful firewall):
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# This single rule allows all return traffic for connections YOU initiated

# Rate limiting (DDoS protection):
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/second --limit-burst 100 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j DROP

# Per-IP connection limit:
iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 50 -j DROP

# Log before dropping:
iptables -A INPUT -j LOG --log-prefix "DROPPED: " --log-level 4
iptables -A INPUT -j DROP

# Block specific country/IP range:
iptables -A INPUT -s 203.0.113.0/24 -j DROP

# Port knocking concept (advanced):
# Require connection to ports 1000, 2000, 3000 in sequence before SSH opens

# Save and restore:
iptables-save > /etc/iptables/rules.v4
iptables-restore < /etc/iptables/rules.v4
```

### nftables (Modern Replacement)

```bash
# nftables is the successor to iptables (kernel 3.13+)
nft list ruleset                          # show all rules
nft add table inet filter                 # create table
nft add chain inet filter input { type filter hook input priority 0 \; }
nft add rule inet filter input tcp dport 22 accept
nft add rule inet filter input tcp dport 80 accept
nft add rule inet filter input ct state established,related accept
nft add rule inet filter input drop
```

### DAY 3 EXERCISES

1. Build a complete firewall ruleset: SSH, HTTP, HTTPS, established, drop rest.
2. Add rate limiting for HTTP: 25 requests/sec per IP.
3. Add connection limiting: max 50 concurrent connections per IP to port 80.
4. Set up logging for dropped packets. Analyze the logs.
5. Block a specific IP range. Verify it's blocked with nc from another host.
6. Compare iptables vs nftables syntax for the same ruleset.
7. Create a firewall script that's idempotent (safe to run multiple times).
8. Test: simulate a brute force attack, verify rate limiting works.

---

## DAY 4: SELinux / AppArmor
**Apr 15, 2025 | 1.5 hrs**

### Mandatory Access Control

```bash
# Standard Linux permissions (DAC — Discretionary Access Control):
#   Owner decides who can access files
#   Root can override everything
#   If app is compromised as root, game over

# SELinux / AppArmor (MAC — Mandatory Access Control):
#   KERNEL enforces rules that even root cannot override
#   Each process has a security context/profile
#   Even if app is compromised, it can only do what policy allows

# ========= SELinux (RHEL/CentOS/Fedora) =========

# Check status:
getenforce                     # Enforcing, Permissive, or Disabled
sestatus                       # detailed status

# Modes:
# Enforcing:  blocks violations (production mode)
# Permissive: logs violations but allows them (debugging mode)
# Disabled:   completely off (NOT recommended)

setenforce 0                   # temporarily set to Permissive
setenforce 1                   # back to Enforcing
# Permanent: edit /etc/selinux/config

# File contexts:
ls -Z /var/www/html/           # show SELinux labels
# Example: system_u:object_r:httpd_sys_content_t:s0

# Troubleshooting (most common SELinux issue):
# "Permission denied" even though regular permissions look fine
# Check: ausearch -m avc --start today    # recent denials
# Fix: restorecon -Rv /var/www/html/      # restore default contexts
# Or: semanage fcontext -a -t httpd_sys_content_t "/custom/web(/.*)?"
#     restorecon -Rv /custom/web/

# SELinux booleans (feature toggles):
getsebool -a | grep httpd      # list httpd-related booleans
setsebool -P httpd_can_network_connect on  # allow httpd to make network connections

# ========= AppArmor (Ubuntu/Debian/SUSE) =========

# Check status:
aa-status                      # show loaded profiles

# Modes:
# enforce: blocks violations
# complain: logs but allows (like SELinux permissive)

# Profile management:
aa-enforce /etc/apparmor.d/usr.sbin.nginx    # set to enforce
aa-complain /etc/apparmor.d/usr.sbin.nginx   # set to complain
aa-disable /etc/apparmor.d/usr.sbin.nginx    # disable profile

# Troubleshooting:
# Check /var/log/syslog or journalctl for "apparmor" denials
# Common fix: edit profile in /etc/apparmor.d/, add needed permissions
```

> **INTERVIEW: "Have you worked with SELinux?"**
>
> Bad answer: "I usually disable it."
> Good answer: "I keep it in Enforcing mode. When I hit permission issues,
> I check ausearch for AVC denials, then fix with restorecon for context
> issues or setsebool for feature toggles. In development I use
> Permissive mode to collect all denials first, then fix them."

### DAY 4 EXERCISES

1. Check if SELinux or AppArmor is active on your system. What mode?
2. If AppArmor: run `aa-status`. How many profiles are loaded/enforced?
3. Simulate a denial: create a web file in a non-standard location. Debug.
4. (SELinux) Use `ausearch -m avc` to find recent denials. Fix one.
5. (AppArmor) Put nginx profile in complain mode, test, re-enforce.
6. List all SELinux booleans related to httpd or nginx.
7. Explain: why is "just disable SELinux" a bad practice?
8. Create/modify a profile to allow a custom application.

---

## DAY 5: PRACTICE LAB — HARDEN A SERVER
**Apr 16, 2025 | 1.5 hrs**

> **TASK: Take a fresh Ubuntu/CentOS server from default to production-hardened.**

```
HARDENING CHECKLIST:

[ ] 1. UPDATE SYSTEM
    apt update && apt upgrade -y
    Enable unattended-upgrades for security patches

[ ] 2. USER MANAGEMENT
    Create non-root admin user with sudo
    Disable root account direct login
    Remove/lock unnecessary user accounts
    Set strong password policy

[ ] 3. SSH HARDENING
    Key-only authentication
    Disable root SSH login
    AllowUsers/AllowGroups
    Non-standard port (optional)
    fail2ban installed and configured

[ ] 4. FIREWALL
    Default deny incoming
    Allow only needed ports (SSH, HTTP/S, monitoring)
    Rate limiting on public ports
    Log dropped traffic

[ ] 5. FILE PERMISSIONS
    Verify /etc/shadow permissions (640)
    Verify /etc/ssh/ permissions
    Remove world-writable files
    Set proper umask (027 or 077)

[ ] 6. SERVICE HARDENING
    Disable unnecessary services
    Each service runs as dedicated user (not root)
    Verify listening ports — only needed ones

[ ] 7. KERNEL HARDENING
    sysctl settings:
    net.ipv4.conf.all.rp_filter = 1         # reverse path filtering
    net.ipv4.conf.all.accept_redirects = 0  # no ICMP redirects
    net.ipv4.conf.all.send_redirects = 0
    net.ipv4.icmp_echo_ignore_broadcasts = 1
    kernel.randomize_va_space = 2            # ASLR enabled
    fs.protected_hardlinks = 1
    fs.protected_symlinks = 1

[ ] 8. LOGGING & AUDITING
    auditd installed and running
    Log rotation configured
    Remote logging (if available)

[ ] 9. MANDATORY ACCESS CONTROL
    SELinux Enforcing / AppArmor profiles loaded

[ ] 10. VERIFICATION
    Run security audit checklist from Day 1
    Compare before and after
    Document all changes made
```

---

## DAY 6: AUDITING & LOGGING
**Apr 17, 2025 | 1.5 hrs**

### auditd — Linux Audit Framework

```bash
# auditd logs security-relevant events at the kernel level
apt install auditd    # or yum install audit

# Check status:
systemctl status auditd
auditctl -l           # list current rules

# Add audit rules:
# Watch a file for changes:
auditctl -w /etc/passwd -p wa -k user_changes
auditctl -w /etc/shadow -p wa -k password_changes
auditctl -w /etc/sudoers -p wa -k sudo_changes
auditctl -w /etc/ssh/sshd_config -p wa -k ssh_changes

# Monitor specific syscalls:
auditctl -a always,exit -F arch=b64 -S execve -k command_execution

# Search audit logs:
ausearch -k user_changes              # by key
ausearch -m USER_LOGIN --start today  # login events today
ausearch -m AVC --start today         # SELinux denials

# Generate reports:
aureport --summary                    # overall summary
aureport --login                      # login report
aureport --failed                     # failed events

# Persistent rules: /etc/audit/rules.d/custom.rules
# -w /etc/passwd -p wa -k user_changes
# -w /etc/shadow -p wa -k password_changes
```

### Log Management

```bash
# Key log files:
/var/log/auth.log     # authentication events (login, sudo, ssh)
/var/log/syslog       # general system messages
/var/log/kern.log     # kernel messages
/var/log/faillog      # failed login attempts
/var/log/lastlog      # last login per user

# Commands:
last                  # recent logins
lastb                 # failed login attempts
lastlog               # last login for all users
who                   # currently logged in users
w                     # who + what they are doing

# Log rotation (/etc/logrotate.d/):
# /var/log/myapp/*.log {
#     daily
#     rotate 30
#     compress
#     delaycompress
#     missingok
#     notifempty
#     postrotate
#         systemctl reload myapp
#     endscript
# }

# Centralized logging (production):
# Ship logs to: ELK stack, Splunk, ClickHouse, Loki
# Why: survive server compromise, correlate across hosts, search at scale
```

### DAY 6 EXERCISES

1. Install and configure auditd. Verify it's running.
2. Add rules to watch /etc/passwd, /etc/shadow, /etc/sudoers.
3. Make a change to /etc/passwd. Find it with ausearch.
4. `aureport --summary` — how many events? Any anomalies?
5. Check auth.log for failed SSH attempts. How many? From where?
6. Set up log rotation for a custom application.
7. `last` and `lastb` — who logged in? Any failed attempts?
8. Configure rsyslog to forward logs to a remote server (or file).

---

## DAY 7: PAM & AUTHENTICATION
**Apr 18, 2025 | 1.5 hrs**

### PAM (Pluggable Authentication Modules)

```bash
# PAM controls HOW authentication works on Linux
# Config files: /etc/pam.d/ (one per service)

# PAM module types:
# auth:     verify identity (password, key, token)
# account:  check if account is allowed (expired? locked?)
# password: handle password changes (complexity, history)
# session:  set up/tear down session (limits, env, logging)

# Password policy (/etc/pam.d/common-password or /etc/security/pwquality.conf):
# minlen = 12           # minimum 12 characters
# minclass = 3          # at least 3 character classes (upper, lower, digit, special)
# maxrepeat = 3         # no more than 3 consecutive same chars
# reject_username       # password cannot contain username
# enforce_for_root      # even root must follow rules

# Account lockout (/etc/pam.d/common-auth):
# pam_faillock.so deny=5 unlock_time=900
# Lock account after 5 failed attempts for 15 minutes

# Commands:
passwd -S username     # account status (locked? expired?)
chage -l username      # password aging info
passwd -l username     # lock account
passwd -u username     # unlock account
chage -M 90 username   # password expires in 90 days
```

### DAY 7 EXERCISES

1. Examine /etc/pam.d/sshd — what modules are used?
2. Set password complexity: minimum 12 chars, 3 classes.
3. Configure account lockout after 5 failed attempts.
4. Check password aging for all users with `chage -l`.
5. Lock a test account. Verify login fails. Unlock.
6. Set password expiry to 90 days for a user.
7. Explain PAM module types: auth, account, password, session.
8. Configure PAM to log all authentication events.

---

## DAY 8: TLS & CERTIFICATES
**Apr 19, 2025 | 1.5 hrs**

### PKI and TLS

```bash
# TLS (Transport Layer Security) encrypts data in transit
# PKI (Public Key Infrastructure) manages trust via certificates

# Certificate chain:
# Root CA (trusted, in browser/OS) -> Intermediate CA -> Server Certificate
# Server proves identity by presenting cert signed by trusted CA

# openssl commands:

# Generate private key:
openssl genrsa -out server.key 4096

# Generate CSR (Certificate Signing Request):
openssl req -new -key server.key -out server.csr

# Self-signed cert (dev/testing only):
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
    -keyout server.key -out server.crt

# Inspect a certificate:
openssl x509 -in server.crt -text -noout

# Check remote server certificate:
openssl s_client -connect google.com:443 -servername google.com
# Shows: certificate chain, expiry, cipher suite

# Check certificate expiry:
openssl s_client -connect example.com:443 2>/dev/null | \
    openssl x509 -noout -dates
# notBefore and notAfter

# Verify certificate chain:
openssl verify -CAfile ca-bundle.crt server.crt

# Let's Encrypt (free certificates):
# certbot certonly --nginx -d example.com
# Auto-renewal: certbot renew (add to cron)
# Certs stored in /etc/letsencrypt/live/example.com/
```

> **CERTIFICATE EXPIRY — #1 PREVENTABLE OUTAGE**
>
> Expired certs cause immediate, total service outages.
> Prevention:
> - Monitor cert expiry with Prometheus (blackbox_exporter)
> - Alert at 30 days, 14 days, 7 days before expiry
> - Automate renewal (Let's Encrypt + certbot)
> - Script: `echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -enddate`

### DAY 8 EXERCISES

1. Generate a self-signed certificate. Inspect it with openssl x509.
2. Check google.com certificate: who issued it? When does it expire?
3. Check 5 of your frequently used sites' cert expiry dates.
4. Set up Let's Encrypt for a test domain (or simulate the flow).
5. Write a script that checks cert expiry for a list of domains, alerts if under 30 days.
6. Explain the certificate chain: Root CA, Intermediate, Server.
7. What happens when a cert expires? What does the user see?
8. Configure nginx with TLS: cert, key, strong cipher suite.

---

## DAY 9: CONTAINER SECURITY
**Apr 21, 2025 | 1.5 hrs**

### Docker/Container Security

```bash
# PRINCIPLE: Containers are NOT security boundaries by default.
# A container with root = root on the host (without proper config).

# 1. Don't run as root:
# In Dockerfile: USER nonroot
# Or: docker run --user 1000:1000

# 2. Read-only filesystem:
# docker run --read-only --tmpfs /tmp

# 3. Drop capabilities:
# docker run --cap-drop ALL --cap-add NET_BIND_SERVICE

# 4. No new privileges:
# docker run --security-opt no-new-privileges

# 5. Resource limits:
# docker run --memory 512m --cpus 1.0

# 6. Image scanning:
# trivy image nginx:latest        # scan for CVEs
# docker scout cves nginx:latest  # Docker Scout

# 7. Use minimal base images:
# FROM alpine:3.19               # 5MB vs 100MB+ for ubuntu
# FROM gcr.io/distroless/base    # even smaller, no shell

# 8. Don't store secrets in images:
# NEVER: COPY secrets.env /app/
# INSTEAD: mount at runtime, use K8s secrets, or vault

# 9. Seccomp profiles (restrict syscalls):
# docker run --security-opt seccomp=profile.json

# 10. Network policies:
# Don't use --network=host in production
# Use Docker networks or K8s NetworkPolicies for isolation
```

### Kubernetes Security Basics

```yaml
# Pod Security Context:
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
```

### DAY 9 EXERCISES

1. Run a container as non-root user. Verify with `whoami` inside.
2. Run with `--read-only`. What breaks? Fix with tmpfs mounts.
3. Run with `--cap-drop ALL`. What can't the process do?
4. Scan an image with trivy. How many CVEs? What severity?
5. Compare image sizes: ubuntu vs alpine vs distroless.
6. Write a secure Dockerfile: non-root, minimal base, no secrets.
7. Apply Pod securityContext in K8s. Verify restrictions.
8. Explain: why are containers NOT security boundaries by default?

---

## DAY 10: CAPSTONE — SECURITY AUDIT & HARDENING
**Apr 22, 2025 | 1.5 hrs**

> **SCENARIO: You join a new team. The senior SRE asks you to audit and harden
> the production web server. Document everything.**

### Phase 1: Audit (30 min)

Run the Day 1 security checklist. Document:
- Users and access (who can log in? sudo?)
- SSH configuration (hardened? keys only?)
- Firewall rules (what's exposed?)
- Listening services (attack surface)
- SUID binaries, world-writable files
- SELinux/AppArmor status
- Patch level (pending security updates?)
- Certificate expiry dates
- Container security (if applicable)

### Phase 2: Harden (40 min)

Apply fixes from the Day 5 hardening checklist:
- SSH hardening + fail2ban
- Firewall with default deny
- Remove unnecessary services
- Fix file permissions
- Enable and configure auditd
- Set up certificate monitoring
- Apply kernel security sysctls

### Phase 3: Verify & Document (20 min)

- Re-run audit. Compare before/after.
- Write a security report:
  - Findings (what was wrong)
  - Changes made (what you fixed)
  - Residual risks (what still needs attention)
  - Recommendations (next steps)
- This document is your interview artifact!

---

> **FILE 05 COMPLETE — LINUX SECURITY & HARDENING**
>
> - Threat models, defense in depth, least privilege, zero trust
> - SSH hardening: key auth, config, fail2ban, best practices
> - Advanced firewalls: rate limiting, connection limiting, nftables
> - SELinux/AppArmor: mandatory access control, troubleshooting
> - Auditing: auditd rules, log management, centralized logging
> - PAM: password policy, account lockout, authentication modules
> - TLS/PKI: certificates, openssl, Let's Encrypt, expiry monitoring
> - Container security: rootless, capabilities, seccomp, image scanning
> - Capstone: full security audit and hardening exercise
>
> **LINUX PHASE COMPLETE! Files 01-05 cover Weeks 1-10 (Feb 15 - Apr 23)**
>
> Next: File S2 — SQL & Databases for SRE (Week 11, Apr 26-30)
