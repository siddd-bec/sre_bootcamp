# LINUX & SYSTEMS — FILE 03 of 48
## LINUX NETWORKING
### TCP/IP, DNS, Sockets, Firewalls, tcpdump, ss, curl, Troubleshooting
**Weeks 5-6 | Mar 15 - Mar 26, 2025 | 10 Days | 1.5 hrs/day**

---

> **WHY NETWORKING IS SRE CORE SKILL #1**
>
> Every production issue eventually becomes a networking issue.
> Apple interview: "Walk me through what happens when you type google.com"
> This file gives you the answer at an expert level.

| Day | Topic | Key Skills |
|-----|-------|-----------|
| 1 | OSI/TCP-IP Model | layers, protocols, encapsulation |
| 2 | IP Addressing & Subnetting | CIDR, private ranges, NAT |
| 3 | TCP Deep Dive | handshake, states, CLOSE_WAIT, TIME_WAIT |
| 4 | DNS | resolution, records, dig, resolv.conf |
| 5 | Network Tools I | ip, ss, ping, traceroute, mtr |
| 6 | Network Tools II | tcpdump, curl, nc (netcat) |
| 7 | Firewalls | iptables chains/rules, nftables, ufw |
| 8 | Load Balancing | L4 vs L7, nginx reverse proxy |
| 9 | Troubleshooting Methodology | 7-step ladder |
| 10 | Capstone | 3 network troubleshooting scenarios |

---

## DAY 1: TCP/IP MODEL
**Mar 15, 2025 | 1.5 hrs**

### TCP/IP 4-Layer Model

```
Layer 4: APPLICATION (HTTP, DNS, SSH, gRPC)     → curl, dig, openssl
Layer 3: TRANSPORT  (TCP, UDP)                   → ss, netstat, tcpdump
Layer 2: INTERNET   (IP, ICMP, ARP)              → ip, ping, traceroute, mtr
Layer 1: LINK       (Ethernet, Wi-Fi)            → ip link, ethtool

Encapsulation: App data → TCP segment → IP packet → Ethernet frame
Debug bottom-up: can you ping? can you connect? can you get response?
```

> **INTERVIEW: WHAT HAPPENS WHEN YOU TYPE google.com**
>
> 1. Browser/OS checks caches, then /etc/hosts, then DNS
> 2. DNS query: resolver → root → .com TLD → google NS → IP
> 3. TCP 3-way handshake: SYN → SYN-ACK → ACK (port 443)
> 4. TLS handshake: ClientHello, certificates, key exchange
> 5. HTTP/2 GET / with Host: google.com
> 6. Server returns HTML (200 OK)
> 7. Browser parses, makes more requests (CSS, JS, images)
> 8. Connection kept alive or closed with FIN
>
> This touches ALL 4 layers. Master it.

**Exercises:** Name layers for HTTP/TCP/IP/ARP/DNS/ICMP, explain encapsulation, DNS over UDP vs TCP, port vs socket, MTU, practice google.com answer aloud.

---

## DAY 2: IP ADDRESSING & SUBNETTING
**Mar 16, 2025 | 1.5 hrs**

```bash
# IPv4: 32-bit = 4 octets. Example: 192.168.1.100/24
# /24 = 256 addresses (254 usable)

# Private ranges (RFC 1918):
# 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
# Special: 127.0.0.1 (loopback), 0.0.0.0 (all), 169.254.x.x (DHCP failed)

ip addr show           # all interfaces
ip route show          # routing table
ip route get 8.8.8.8   # which interface for this IP
ip neigh show          # ARP table
```

> **SUBNETTING CHEAT SHEET**
> /32=1, /31=2, /30=4, /28=16, /27=32, /26=64, /25=128, /24=256
> Formula: hosts = 2^(32-prefix) - 2

**Exercises:** Subnet masks, usable hosts in /26, same-subnet checks, ip addr/route analysis, 169.254.x.x diagnosis.

---

## DAY 3: TCP DEEP DIVE
**Mar 17, 2025 | 1.5 hrs**

```
3-WAY HANDSHAKE: SYN → SYN-ACK → ACK (connection open)
4-WAY TEARDOWN:  FIN → ACK → FIN → ACK (connection close)

TCP STATES:
  LISTEN       — server waiting for connections
  ESTABLISHED  — data flowing
  TIME_WAIT    — normal after close (2*MSL ~60s)
  CLOSE_WAIT   — received FIN, app hasn't closed = BUG!
  SYN_RECV     — many = possible SYN flood

ss -s                        # summary of all states
ss -tn state established     # active connections
ss -tn state close-wait      # find stuck connections
```

> **CLOSE_WAIT = APPLICATION BUG**
>
> Remote side closed, YOUR side hasn't called close().
> Always an app bug: unclosed HTTP responses, DB connection leaks.
> Diagnosis: `ss -tnp state close-wait` (shows which process).
> Fix: application code must close connections in try/finally.

**Exercises:** ss -s states, count per remote IP, listening services, explain TIME_WAIT vs CLOSE_WAIT, TCP retransmissions, draw handshake from memory.

---

## DAY 4: DNS
**Mar 18, 2025 | 1.5 hrs**

```bash
# Resolution order: /etc/hosts → /etc/resolv.conf → DNS server

# Record types: A (IPv4), AAAA (IPv6), CNAME (alias), MX (mail),
# NS (nameserver), TXT (SPF/DKIM), SRV (K8s service discovery)

dig google.com A +short       # just the IP
dig google.com MX              # mail servers
dig @8.8.8.8 google.com       # query specific server
dig google.com +trace          # full resolution path
dig -x 142.250.80.46           # reverse lookup

# /etc/resolv.conf:
# nameserver 8.8.8.8          # primary DNS
# search mycompany.com        # appended to short names
# options ndots:5             # K8s uses this (can cause slowness!)

# Common issues:
# 1. Wrong resolv.conf (overwritten by DHCP)
# 2. DNS server unreachable (firewall blocking port 53)
# 3. Stale cache (old IP after migration)
# 4. ndots too high in K8s (extra lookups = slow)
```

**Exercises:** dig domain records, +trace, resolv.conf inspection, /etc/hosts override, DNS latency measurement, simulate DNS failure.

---

## DAYS 5-6: NETWORK TOOLS
**Mar 19-20, 2025**

### Diagnostic Tools

```bash
ss -tlnp          # listening TCP + process
ss -ulnp          # listening UDP
ss -s             # summary by state
ping -c4 host     # reachability
traceroute host   # path, hop by hop
mtr --report host # live traceroute with loss% and latency
```

### tcpdump — Packet Capture

```bash
tcpdump -i eth0 -nn port 80        # HTTP, no DNS resolve
tcpdump -i eth0 host 10.0.0.5      # specific host
tcpdump -i any port 53              # DNS all interfaces
tcpdump -nn 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # handshakes
tcpdump -w capture.pcap             # save to file
tcpdump -r capture.pcap             # read from file
```

### curl & nc

```bash
curl -v https://api.example.com                    # verbose TLS + headers
curl -o /dev/null -s -w '%{http_code} %{time_total}s' https://google.com
nc -zv host 80                                     # check port open
nc -zv host 80-90                                  # scan range
```

**Exercises (16):** ip addr, ss mapping, ping MTU test, traceroute comparison, mtr slowest hop, tcpdump handshake/DNS capture, curl timing, nc port checks, retransmission capture.

---

## DAY 7: FIREWALLS
**Mar 21, 2025 | 1.5 hrs**

```bash
# iptables: INPUT/OUTPUT/FORWARD chains, top-to-bottom, first match wins
iptables -L -n -v                    # list rules
iptables -A INPUT -p tcp --dport 22 -j ACCEPT   # allow SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT   # allow HTTP
iptables -A INPUT -p tcp --dport 443 -j ACCEPT  # allow HTTPS
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -j DROP                        # deny rest

# CRITICAL: SSH rule MUST be ABOVE DROP rule!

# ufw (simpler):
ufw enable && ufw allow 22/tcp && ufw allow 80/tcp
```

> **ORDER MATTERS:** WRONG: DROP first, ACCEPT after (never reached). RIGHT: ACCEPT ports, ESTABLISHED, then DROP.

**Exercises:** List rules, create allow/deny set, test blocked port, subnet-only access, log drops, rate limit SSH, ufw setup, SSH lockout recovery plan.

---

## DAY 8: LOAD BALANCING & PROXIES
**Mar 22, 2025 | 1.5 hrs**

```
L4 (Transport): routes by IP+port. Fast, protocol-agnostic.
L7 (Application): routes by URL/headers. SSL termination, URL routing.

# nginx reverse proxy:
upstream backend {
    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=1;
}
server {
    listen 80;
    location / { proxy_pass http://backend; }
    location /api/ { proxy_pass http://api-backend; }
}

Health checks: active (periodic /health) + passive (failure count)
```

**Exercises:** nginx reverse proxy, weighted routing, health check failover, URL routing, L4 vs L7, SSL termination, sticky sessions.

---

## DAY 9: TROUBLESHOOTING METHODOLOGY
**Mar 24, 2025**

### The 7-Step Network Debugging Ladder

```
Start at bottom, work up:
1. Interface up?      → ip link show
2. Have IP?           → ip addr show (169.254 = DHCP failed)
3. Reach gateway?     → ping gateway_ip
4. Reach destination? → ping dest / traceroute dest
5. DNS resolving?     → dig hostname
6. Port open?         → nc -zv host port / ss -tlnp
7. App responding?    → curl -v http://host/health

If it fails at step N, the problem is at layer N. Don't skip!
```

> **TOP 10 NETWORK ISSUES FOR SRE**
> 1. DNS failure (wrong resolv.conf)
> 2. Firewall blocking (iptables/security group)
> 3. Service not listening (crashed, wrong bind addr)
> 4. Connection timeout (network path, firewall)
> 5. Connection refused (service down, wrong port)
> 6. TLS cert expired or mismatch
> 7. MTU mismatch (silent packet drops)
> 8. TCP connection leak (CLOSE_WAIT)
> 9. DNS TTL too long (stale IPs)
> 10. LB misconfigured (wrong backend, no health checks)

---

## DAY 10: CAPSTONE — 3 TROUBLESHOOTING SCENARIOS
**Mar 25, 2025**

**Scenario 1: Web App Unreachable (30 min)** — Walk the 7-step ladder: DNS → ping → port → service → config → TLS → logs

**Scenario 2: Intermittent Latency (30 min)** — mtr (lossy hops?), tcpdump (retransmits?), ss (CLOSE_WAIT?), curl timing, LB balance

**Scenario 3: Service-to-Service Failure (30 min)** — ping, nc port, ss -tlnp on target, firewalls both sides, DNS cache

---

> **FILE 03 COMPLETE — LINUX NETWORKING**
> Next: File 04 — Linux Performance (Weeks 7-8, Mar 29 - Apr 9)
