# LINUX & SYSTEMS — FILE 03 of 48
## LINUX NETWORKING
### TCP/IP, DNS, Sockets, Firewalls, tcpdump, ss, curl, Troubleshooting
**Weeks 5-6 | Mar 15 - Mar 26, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY NETWORKING IS SRE CORE SKILL #1**
>
> Every production issue eventually becomes a networking issue.
> "The app is slow" → DNS? Firewall? Connection leak? TLS expired?
>
> Apple interview: "Walk me through what happens when you type google.com"
> This file gives you the answer at an expert level.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | TCP/IP Model | layers, protocols, encapsulation, ports |
| 2 | IP Addressing & Subnetting | CIDR, private ranges, NAT, routing |
| 3 | TCP Deep Dive | handshake, states, CLOSE_WAIT, TIME_WAIT |
| 4 | DNS | resolution, records, dig, resolv.conf, caching |
| 5 | Network Tools I | ip, ss, ping, traceroute, mtr |
| 6 | Network Tools II | tcpdump, curl, nc (netcat) |
| 7 | Firewalls | iptables chains, rules, nftables, ufw |
| 8 | Load Balancing & Proxies | L4 vs L7, nginx reverse proxy, health checks |
| 9 | Troubleshooting Methodology | 7-step debugging ladder |
| 10 | Capstone | 3 network troubleshooting scenarios |

---

## DAY 1: TCP/IP MODEL
**Mar 15, 2025 | 1.5 hrs**

> 00:00-00:20 Teach: layers, protocols, encapsulation
> 00:20-01:10 Hands-on: identify protocols at each layer
> 01:10-01:30 Drills: layer identification exercises

### The TCP/IP 4-Layer Model

```
Layer 4: APPLICATION    HTTP, HTTPS, DNS, SSH, gRPC, SMTP, FTP
         What?          The actual data/protocol apps use
         Tools:         curl, dig, openssl, wget

Layer 3: TRANSPORT      TCP, UDP
         What?          Reliable (TCP) or fast (UDP) delivery
         Tools:         ss, netstat, tcpdump

Layer 2: INTERNET       IP, ICMP, ARP
         What?          Addressing and routing between networks
         Tools:         ip, ping, traceroute, mtr

Layer 1: LINK           Ethernet, Wi-Fi, ARP
         What?          Physical frame delivery on local network
         Tools:         ip link, ethtool, arp

HOW DATA FLOWS (encapsulation):
  App creates data
    → TCP wraps it in a SEGMENT (adds port numbers)
      → IP wraps it in a PACKET (adds source/dest IP)
        → Ethernet wraps it in a FRAME (adds MAC addresses)
          → Bits on the wire

DEBUG BOTTOM-UP:
  1. Can you reach the network? (Layer 1-2: ip link, ping gateway)
  2. Can you reach the host? (Layer 2: ping destination)
  3. Can you connect to the port? (Layer 3: nc, ss)
  4. Does the app respond? (Layer 4: curl, dig)
```

### INTERVIEW: What Happens When You Type google.com

```
This is the MOST COMMON network interview question.
It touches ALL 4 layers. Practice saying it aloud.

1. BROWSER/OS: Check caches → /etc/hosts → DNS resolver

2. DNS RESOLUTION (Layer 4 over UDP):
   Your resolver → Root server (.) → .com TLD → Google's NS → IP address
   dig google.com +trace shows this chain

3. TCP 3-WAY HANDSHAKE (Layer 3):
   Client → SYN → Server (port 443)
   Client ← SYN-ACK ← Server
   Client → ACK → Server
   Connection established

4. TLS HANDSHAKE (Layer 4):
   ClientHello (supported ciphers)
   ServerHello + Certificate
   Key exchange → Encrypted tunnel ready

5. HTTP/2 REQUEST (Layer 4):
   GET / HTTP/2.0
   Host: google.com

6. RESPONSE:
   200 OK + HTML content

7. RENDERING:
   Browser parses HTML, makes more requests (CSS, JS, images)
   Each may reuse the TCP connection (keep-alive)

8. CONNECTION CLOSE:
   FIN → ACK → FIN → ACK (or kept alive for reuse)
```

### Common Ports (Memorize These)

```
Port    Protocol    Service
22      TCP         SSH
53      TCP/UDP     DNS
80      TCP         HTTP
443     TCP         HTTPS
3306    TCP         MySQL
5432    TCP         PostgreSQL
6379    TCP         Redis
8080    TCP         HTTP alt (common for apps)
9090    TCP         Prometheus
9100    TCP         Node Exporter
27017   TCP         MongoDB
2379    TCP         etcd
```

### DAY 1 EXERCISES

> **EXERCISE 1:** For each protocol (HTTP, TCP, IP, ARP, DNS, ICMP, SSH), name its layer.
> **EXERCISE 2:** Explain encapsulation: what gets added at each layer?
> **EXERCISE 3:** Why does DNS use UDP? When does it fall back to TCP?
> **EXERCISE 4:** What's the difference between a port and a socket?
> **EXERCISE 5:** Practice the "what happens when you type google.com" answer aloud. Time yourself — aim for 60 seconds.

---

## DAY 2: IP ADDRESSING & SUBNETTING
**Mar 16, 2025 | 1.5 hrs**

### IPv4 Addressing

```bash
# IPv4: 32-bit address = 4 octets (each 0-255)
# Example: 192.168.1.100

# CIDR notation: IP/prefix_length
# /24 = 256 addresses (254 usable — network + broadcast reserved)
# /16 = 65,536 addresses
# /8  = 16,777,216 addresses

# PRIVATE RANGES (RFC 1918 — NOT routable on internet):
# 10.0.0.0/8       (10.x.x.x)        — large networks
# 172.16.0.0/12    (172.16-31.x.x)   — medium networks
# 192.168.0.0/16   (192.168.x.x)     — home/small office

# SPECIAL ADDRESSES:
# 127.0.0.1        loopback (localhost)
# 0.0.0.0          "all interfaces" (listen on everything)
# 255.255.255.255  broadcast
# 169.254.x.x      link-local (DHCP failed! Auto-assigned)
```

### Subnetting Cheat Sheet

```
Prefix  Subnet Mask       Addresses  Usable Hosts
/32     255.255.255.255   1          0 (single host)
/31     255.255.255.254   2          2 (point-to-point)
/30     255.255.255.252   4          2
/29     255.255.255.248   8          6
/28     255.255.255.240   16         14
/27     255.255.255.224   32         30
/26     255.255.255.192   64         62
/25     255.255.255.128   128        126
/24     255.255.255.0     256        254
/16     255.255.0.0       65,536     65,534
/8      255.0.0.0         16.7M      16.7M

Formula: usable hosts = 2^(32 - prefix) - 2
```

### IP Commands

```bash
ip addr show                        # all interfaces and IPs
ip addr show eth0                   # specific interface
ip route show                       # routing table
ip route get 8.8.8.8               # which interface routes to this IP
ip neigh show                       # ARP table (IP → MAC mappings)
ip link show                        # interface status (up/down)
ip -s link show eth0               # interface statistics (errors, drops)
```

### DAY 2 EXERCISES

> **EXERCISE 1:** How many usable hosts in /26? /28? /30?
> **EXERCISE 2:** Are these on the same subnet? 10.0.1.50/24 and 10.0.1.200/24? What about 10.0.1.50/24 and 10.0.2.50/24?
> **EXERCISE 3:** Run `ip addr show`. Identify each interface, IP, and subnet.
> **EXERCISE 4:** Run `ip route show`. What is the default gateway? Why does it matter?
> **EXERCISE 5:** A server gets a 169.254.x.x address. What happened? (DHCP failed)

---

## DAY 3: TCP DEEP DIVE
**Mar 17, 2025 | 1.5 hrs**

### TCP Connection Lifecycle

```
3-WAY HANDSHAKE (connection open):
  Client → SYN       → Server    "I want to connect"
  Client ← SYN-ACK   ← Server    "OK, acknowledged"
  Client → ACK        → Server    "Connection established"

DATA TRANSFER:
  Sequence numbers track every byte sent
  Acknowledgments confirm receipt
  Retransmission if ACK not received (timeout)

4-WAY TEARDOWN (connection close):
  Client → FIN       → Server    "I'm done sending"
  Client ← ACK       ← Server    "Got it"
  Client ← FIN       ← Server    "I'm done too"
  Client → ACK        → Server    "Goodbye"
```

### TCP States — What They Mean for SRE

```
State           Meaning                      SRE Concern?
─────────────   ─────────────────────────    ────────────
LISTEN          Waiting for connections       Normal (servers)
SYN_SENT        Connecting to remote          Many = remote down/blocked
SYN_RECV        Received SYN, sent SYN-ACK   Many = SYN flood attack!
ESTABLISHED     Active data transfer          Normal
FIN_WAIT_1      Sent FIN, waiting for ACK     Transient
FIN_WAIT_2      FIN ACKed, waiting for FIN    Should clear quickly
TIME_WAIT       Waiting after close (2×MSL)   NORMAL — prevents packet confusion
CLOSE_WAIT      Received FIN, haven't closed  ⚠️ APPLICATION BUG — VERY COMMON
LAST_ACK        Sent FIN after CLOSE_WAIT     Transient
CLOSING         Both sides FIN simultaneously Rare
```

### CLOSE_WAIT — The SRE's Nemesis

```
CLOSE_WAIT means:
  "The REMOTE side closed the connection, but YOUR APP hasn't called close()."

This is ALWAYS an application code bug:
  - Unclosed HTTP connections
  - Database connection pool leaks
  - Not closing in try/finally

Diagnosis:
  ss -tnp state close-wait
  # Shows which process has the leaking connections

  # Count per process:
  ss -tnp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn

Fix:
  - Fix the application code (add try/finally close)
  - As temporary relief: restart the process

INTERVIEW ANSWER:
  "CLOSE_WAIT means the remote side closed but our app didn't call close().
   It's always an application bug — usually a connection not being closed in
   a finally block. I'd check with ss -tnp state close-wait to find the
   leaking process, then fix the code."
```

### TIME_WAIT — Normal but Can Be Excessive

```bash
# TIME_WAIT is NORMAL — it lasts ~60 seconds after close.
# It prevents old packets from confusing new connections on same port.
# But too many can exhaust ephemeral ports (only 28,000 by default).

# Check:
ss -s                                # summary of all states
ss -tn state time-wait | wc -l      # count TIME_WAITs

# If excessive (>10,000):
# Tune (sysctl):
net.ipv4.tcp_tw_reuse = 1           # allow reuse of TIME_WAIT sockets
net.ipv4.ip_local_port_range = 1024 65535   # more ephemeral ports
```

### Monitoring TCP States

```bash
# Summary of all states:
ss -s

# Count by state:
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# Example output:
#  150 ESTAB         ← active connections
#   45 TIME-WAIT     ← recently closed (normal)
#   12 CLOSE-WAIT    ← ⚠️ APP BUG — investigate!
#    3 LISTEN        ← servers waiting
#    1 SYN-SENT      ← connecting

# Established connections per remote IP:
ss -tn state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# Find connections to a specific port:
ss -tn state established '( dport = :5432 )'    # who's connected to PostgreSQL?
```

### DAY 3 EXERCISES

> **EXERCISE 1:** Run `ss -s`. Document each state and count. Any concerns?
> **EXERCISE 2:** Count connections per remote IP. Any single IP with too many connections?
> **EXERCISE 3:** Check for CLOSE_WAIT. If found, which process owns them?
> **EXERCISE 4:** Explain TIME_WAIT vs CLOSE_WAIT. Which is normal? Which is a bug?
> **EXERCISE 5:** Draw the TCP 3-way handshake and 4-way teardown from memory.
> **EXERCISE 6:** What does high SYN_RECV count indicate? (SYN flood / DDoS)

---

## DAY 4: DNS
**Mar 18, 2025 | 1.5 hrs**

### DNS Resolution Chain

```
When you access web-01.prod.company.com:

1. Local cache (OS, browser)
2. /etc/hosts file (manual overrides)
3. /etc/resolv.conf → DNS server IP
4. Recursive resolver → Root (.) → .com → company.com → prod.company.com → A record

Each step can fail = each step is a debugging point.
```

### DNS Record Types

```
A       IPv4 address                google.com → 142.250.80.46
AAAA    IPv6 address                google.com → 2607:f8b0:4004:...
CNAME   Alias (points to another name)  www.google.com → google.com
MX      Mail server                 google.com → smtp.google.com (priority 10)
NS      Nameserver                  google.com → ns1.google.com
TXT     Text data (SPF, DKIM, verification)
SRV     Service discovery           _http._tcp.example.com → server:port
PTR     Reverse lookup (IP → name)  142.250.80.46 → google.com
SOA     Start of Authority          Zone admin info, serial, TTL
```

### dig — The DNS Swiss Army Knife

```bash
# Basic lookup:
dig google.com                           # full output
dig google.com A +short                  # just the IP(s)
dig google.com AAAA +short               # IPv6
dig google.com MX                        # mail servers
dig google.com NS                        # nameservers
dig google.com TXT                       # TXT records

# Query specific DNS server:
dig @8.8.8.8 google.com                 # ask Google's DNS
dig @1.1.1.1 google.com                 # ask Cloudflare's DNS

# Trace full resolution path:
dig google.com +trace
# Shows: root → .com → google.com → answer
# INCREDIBLY useful for debugging "DNS works from X but not Y"

# Reverse lookup:
dig -x 142.250.80.46                     # IP → hostname

# Check TTL (time to live):
dig google.com | grep -E "^google"
# google.com.  203  IN  A  142.250.80.46
#              ^^^ TTL in seconds (how long to cache)
```

### DNS Configuration

```bash
# /etc/resolv.conf — where to find DNS:
cat /etc/resolv.conf
# nameserver 8.8.8.8            # primary DNS server
# nameserver 8.8.4.4            # backup DNS server
# search mycompany.com          # append to short names
# options ndots:5               # K8s uses this!

# /etc/hosts — manual overrides (checked BEFORE DNS):
cat /etc/hosts
# 127.0.0.1  localhost
# 10.0.1.50  web-prod            # local shortcut

# Resolution order (/etc/nsswitch.conf):
grep hosts /etc/nsswitch.conf
# hosts: files dns              # files (/etc/hosts) first, then DNS
```

### Common DNS Issues for SRE

```bash
# 1. Wrong resolv.conf (overwritten by DHCP or NetworkManager):
cat /etc/resolv.conf             # verify correct nameservers

# 2. DNS server unreachable:
dig @10.0.0.1 google.com        # test specific server
# If timeout → firewall blocking port 53, or server down

# 3. Stale cache (old IP after migration):
# Clear local cache:
sudo systemd-resolve --flush-caches   # systemd
sudo service nscd restart              # nscd

# 4. K8s ndots issue (too many lookups):
# ndots:5 means any name with < 5 dots gets search suffixes appended
# "api-server" becomes: api-server.default.svc.cluster.local, then others
# Fix: use FQDN with trailing dot: "api-server.default.svc.cluster.local."

# 5. TTL too high (changes don't propagate):
dig mysite.com | grep TTL        # check current TTL
# Lower TTL BEFORE migrations. Wait old TTL. Then migrate.
```

### DAY 4 EXERCISES

> **EXERCISE 1:** Use dig to find A, AAAA, MX, NS, TXT records for a domain.
> **EXERCISE 2:** Use `dig +trace` to see the full resolution path. Explain each step.
> **EXERCISE 3:** Check /etc/resolv.conf and /etc/hosts. What's configured?
> **EXERCISE 4:** Add an entry to /etc/hosts. Verify it overrides DNS.
> **EXERCISE 5:** Measure DNS lookup time: `dig google.com | grep "Query time"`
> **EXERCISE 6:** Simulate DNS failure: what happens if resolv.conf points to a wrong IP?

---

## DAYS 5-6: NETWORK TOOLS
**Mar 19-20, 2025 | 1.5 hrs each**

### ss — Socket Statistics (replacement for netstat)

```bash
ss -tlnp                         # TCP, Listening, Numeric, Process
ss -ulnp                         # UDP, Listening, Numeric, Process
ss -s                            # summary (count by state)
ss -tn                           # all TCP connections (no header)
ss -tn state established         # only established
ss -tn state close-wait          # only CLOSE_WAIT (find leaks!)
ss -tn '( dport = :443 )'       # connections to port 443
ss -tn '( sport = :8080 )'      # connections FROM port 8080
```

### ping, traceroute, mtr

```bash
ping -c 4 google.com             # 4 packets
ping -c 4 -s 1472 google.com    # test MTU (no fragment)

traceroute google.com            # show path hop by hop
traceroute -n google.com         # numeric (no DNS, faster)

mtr google.com                   # BEST: live traceroute with loss% + latency
mtr --report -c 100 google.com  # report mode (100 packets)
# Look for: high loss% at a specific hop, high latency jump
```

### tcpdump — Packet Capture

```bash
# tcpdump sees ACTUAL packets on the wire. The ultimate debugging tool.

tcpdump -i eth0 -nn port 80              # HTTP traffic, no DNS resolution
tcpdump -i eth0 -nn host 10.0.0.5        # all traffic to/from host
tcpdump -i any port 53                    # DNS on all interfaces
tcpdump -i eth0 -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # handshakes + closes
tcpdump -w capture.pcap -i eth0 port 443 # save to file (open in Wireshark)
tcpdump -r capture.pcap                   # read saved capture
tcpdump -c 100 -i eth0                    # capture only 100 packets

# SRE PATTERNS:
# See if traffic is reaching the server:
tcpdump -i eth0 -nn port 8080 -c 10
# See DNS queries:
tcpdump -i any port 53 -nn
# Capture TCP handshake:
tcpdump -i eth0 -nn 'tcp[tcpflags] == tcp-syn'
```

### curl — HTTP Client

```bash
curl https://api.example.com                         # basic GET
curl -v https://api.example.com                      # verbose (see TLS, headers)
curl -I https://api.example.com                      # HEAD only (headers)
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" URL

# TIMING — essential for SRE:
curl -o /dev/null -s -w "\
    DNS:      %{time_namelookup}s\n\
    Connect:  %{time_connect}s\n\
    TLS:      %{time_appconnect}s\n\
    Start:    %{time_starttransfer}s\n\
    Total:    %{time_total}s\n\
    Status:   %{http_code}\n" https://google.com

# Check certificate:
curl -vI https://example.com 2>&1 | grep -E "expire|subject|issuer"
# Or: openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -dates
```

### nc (netcat) — TCP/UDP Swiss Army Knife

```bash
nc -zv host 80                    # check single port
nc -zv host 80-90                 # check port range
nc -zv -w3 host 5432             # with 3s timeout
```

### DAY 5-6 EXERCISES

> **EXERCISE 1:** Use ss to: list all listening ports, count connections by state, find CLOSE_WAIT.
> **EXERCISE 2:** Run mtr to google.com. Identify the hop with highest latency. Any packet loss?
> **EXERCISE 3:** Capture a TCP handshake with tcpdump. Identify SYN, SYN-ACK, ACK.
> **EXERCISE 4:** Capture DNS queries with tcpdump. Which domains is your server looking up?
> **EXERCISE 5:** Use curl timing to measure DNS, connect, TLS, and total time for 3 different URLs.
> **EXERCISE 6:** Use nc to check if ports 22, 80, 443, 5432, 6379 are open on a server.

---

## DAY 7: FIREWALLS
**Mar 21, 2025 | 1.5 hrs**

### iptables — The Foundation

```bash
# iptables processes packets through CHAINS:
# INPUT    — packets TO this server
# OUTPUT   — packets FROM this server
# FORWARD  — packets routed THROUGH this server

# Rules are processed TOP TO BOTTOM — first match wins!

# View rules:
iptables -L -n -v                             # list all with counters
iptables -L INPUT -n --line-numbers           # INPUT chain with numbers

# Basic secure setup:
iptables -A INPUT -i lo -j ACCEPT                                    # allow loopback
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT     # allow existing
iptables -A INPUT -p tcp --dport 22 -j ACCEPT                       # allow SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT                       # allow HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT                      # allow HTTPS
iptables -A INPUT -p tcp --dport 9100 -j ACCEPT                     # node_exporter
iptables -A INPUT -j DROP                                             # drop everything else

# ⚠️ CRITICAL: SSH rule MUST come BEFORE DROP rule!
# Wrong order = you lock yourself out.

# Delete a rule:
iptables -D INPUT 5                           # delete rule #5
iptables -D INPUT -p tcp --dport 80 -j ACCEPT # delete by specification

# Rate limit SSH (prevent brute force):
iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/min --limit-burst 5 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
```

### ufw — Simplified Firewall

```bash
# ufw wraps iptables with simpler commands:
sudo ufw enable                              # turn on
sudo ufw status verbose                      # view rules
sudo ufw allow 22/tcp                        # allow SSH
sudo ufw allow 80/tcp                        # allow HTTP
sudo ufw allow 443/tcp                       # allow HTTPS
sudo ufw allow from 10.0.0.0/8 to any port 5432   # PostgreSQL from internal only
sudo ufw deny 3306                           # block MySQL
sudo ufw delete allow 80/tcp                 # remove rule
```

### DAY 7 EXERCISES

> **EXERCISE 1:** Create a firewall rule set: allow SSH, HTTP, HTTPS, node_exporter. Drop all else.
> **EXERCISE 2:** Test a blocked port (try to connect with nc). Verify it's blocked.
> **EXERCISE 3:** Set up ufw on a server. Test with nc from another machine.
> **EXERCISE 4:** Why must the SSH ACCEPT rule come before the DROP rule?
> **EXERCISE 5:** How would you recover if you accidentally blocked SSH? (console access, timeout rule)

---

## DAY 8: LOAD BALANCING & PROXIES
**Mar 22, 2025 | 1.5 hrs**

### L4 vs L7 Load Balancing

```
L4 (Transport Layer):
  Routes by IP + port. Fast. Protocol-agnostic.
  Can't inspect HTTP content.
  Example: HAProxy in TCP mode, AWS NLB.

L7 (Application Layer):
  Routes by URL, headers, cookies. Can do SSL termination.
  Slower but much more flexible.
  Example: nginx, HAProxy in HTTP mode, AWS ALB.

USE L4: Raw TCP, database connections, gRPC
USE L7: HTTP routing, SSL termination, URL-based routing
```

### nginx as Reverse Proxy

```nginx
# /etc/nginx/conf.d/app.conf

upstream backend {
    server 10.0.1.10:8080 weight=3;    # gets 3x traffic
    server 10.0.1.11:8080 weight=1;    # gets 1x traffic
    server 10.0.1.12:8080 backup;      # only when others fail
}

upstream api_backend {
    server 10.0.2.10:9090;
    server 10.0.2.11:9090;
}

server {
    listen 80;
    server_name myapp.com;

    # URL-based routing (L7):
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://api_backend;
    }

    location /health {
        return 200 "OK\n";
    }
}
```

### Health Check Patterns

```
ACTIVE health checks (nginx/HAProxy polls backends):
  Every N seconds, hit /health on each backend.
  If down → remove from pool. If back → re-add.

PASSIVE health checks (count failures):
  If backend returns 5 errors in a row → mark unhealthy.
  Cheaper but slower to detect failures.

SRE BEST PRACTICE: Use both.
  Active: catch dead servers quickly
  Passive: catch intermittent failures
```

### DAY 8 EXERCISES

> **EXERCISE 1:** Write nginx config with 3 backends, weighted routing, and a backup server.
> **EXERCISE 2:** Add URL routing: / → web backend, /api/ → api backend.
> **EXERCISE 3:** Explain: when should you use L4 vs L7 load balancing?
> **EXERCISE 4:** What happens when a backend fails? How does nginx handle it?
> **EXERCISE 5:** What is SSL termination? Why do it at the load balancer?

---

## DAY 9: TROUBLESHOOTING METHODOLOGY
**Mar 24, 2025 | 1.5 hrs**

### The 7-Step Network Debugging Ladder

```
Start at the BOTTOM. Work UP. If it fails at step N, the problem is at layer N.

Step 1: Interface up?        ip link show
                             Is the interface UP? Link detected?

Step 2: Have IP address?     ip addr show
                             Got an IP? Or 169.254.x.x (DHCP failed)?

Step 3: Reach gateway?       ping $(ip route | grep default | awk '{print $3}')
                             Can we reach our own gateway?

Step 4: Reach destination?   ping dest_ip / traceroute dest_ip
                             Is the path clear? Where does it fail?

Step 5: DNS resolving?       dig hostname
                             Does the name resolve to an IP?

Step 6: Port open?           nc -zv host port / ss -tlnp
                             Is the service listening? Firewall blocking?

Step 7: App responding?      curl -v http://host/health
                             Does the application return valid response?

If Step 1 fails → physical/driver issue
If Step 3 fails → local network config
If Step 4 fails → routing/firewall between networks
If Step 5 fails → DNS configuration
If Step 6 fails → service not running or firewall
If Step 7 fails → application bug
```

### Top 10 Network Issues for SRE

```
1.  DNS failure           Wrong resolv.conf, DNS server down, stale cache
2.  Firewall blocking     iptables DROP, security group, ACL
3.  Service not listening Crashed, wrong bind address (127.0.0.1 vs 0.0.0.0)
4.  Connection timeout    Network path blocked, firewall, routing
5.  Connection refused    Service down, wrong port
6.  TLS cert expired      Certificate not renewed, wrong SNI
7.  MTU mismatch          Silent packet drops (esp. VPN/tunnel)
8.  Connection leak       CLOSE_WAIT piling up (app bug)
9.  DNS TTL too long      Stale IPs after migration
10. LB misconfigured      Wrong backend, no health checks, stale config
```

### DAY 9 EXERCISES

> **EXERCISE 1:** Practice the 7-step ladder on a working service. Document each step.
> **EXERCISE 2:** For each of the top 10 issues, write: what command detects it? How to fix?
> **EXERCISE 3:** A web app returns "connection refused". Walk through debugging steps.

---

## DAY 10: CAPSTONE — 3 TROUBLESHOOTING SCENARIOS
**Mar 25-26, 2025 | 1.5 hrs each**

> **SCENARIO 1: Web Application Unreachable (30 min)**
>
> Users report "site is down." Walk the 7-step ladder:
> a) Check interface, IP, gateway
> b) Ping the server from different locations
> c) Check DNS resolution
> d) Check if the port is open (ss, nc)
> e) Check the service (systemctl, logs)
> f) Check the firewall
> g) Check the application (curl -v, access logs)

> **SCENARIO 2: Intermittent Latency Spikes (30 min)**
>
> Response times spike every few minutes:
> a) mtr to check path quality (packet loss? latency?)
> b) tcpdump for retransmissions
> c) ss for connection state issues (CLOSE_WAIT?)
> d) curl timing breakdown (DNS? TLS? backend?)
> e) Check load balancer distribution

> **SCENARIO 3: Service-to-Service Communication Failure (30 min)**
>
> Service A can't reach Service B:
> a) Can A ping B?
> b) Can A reach B's port? (nc -zv)
> c) Is B actually listening? (ss -tlnp on B)
> d) Firewall rules on BOTH sides
> e) DNS resolution from A's perspective
> f) tcpdump on both sides simultaneously

---

## WEEKS 5-6 SUMMARY

> **FILE 03 COMPLETE — LINUX NETWORKING**
>
> ✓ TCP/IP model: 4 layers, encapsulation, common ports
> ✓ IP addressing: CIDR, subnetting, private ranges
> ✓ TCP deep dive: handshake, states, CLOSE_WAIT, TIME_WAIT
> ✓ DNS: records, dig, resolv.conf, caching, common issues
> ✓ Network tools: ss, ping, traceroute, mtr, tcpdump, curl, nc
> ✓ Firewalls: iptables, ufw, rule ordering
> ✓ Load balancing: L4 vs L7, nginx, health checks
> ✓ Troubleshooting: 7-step ladder, top 10 issues
> ✓ 50+ exercises including 3 troubleshooting scenarios
>
> **INTERVIEW PREP:**
> - "What happens when you type google.com?" → Full DNS→TCP→TLS→HTTP flow
> - "What is CLOSE_WAIT?" → Remote closed, app hasn't. Always an app bug.
> - "How do you debug a network issue?" → 7-step ladder, bottom up
> - "What's the difference between L4 and L7 LB?" → L4=IP+port, L7=HTTP content
> - "Explain TCP handshake" → SYN → SYN-ACK → ACK
>
> Next: File 04 — Linux Performance (Weeks 7-8, Mar 29 - Apr 9)
