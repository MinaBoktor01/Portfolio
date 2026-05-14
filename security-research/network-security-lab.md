
**Difficulty:** Intermediate
**Environment:** SEED Labs (Docker Network Environment)

---
## Overview
This lab explores core network security concepts including packet sniffing, ICMP spoofing, custom traceroute implementation, iptables firewall configuration, stateful connection tracking, traffic limiting, and load balancing. All tasks were performed in a controlled Docker-based lab environment.

**Tools & Technologies:** Scapy · iptables · conntrack · Python3 · Docker networking

---

## Network Topology

```
Attacker (10.9.0.1) ←→ Router ←→ Internal Hosts (10.9.0.5, 10.9.0.6)
                                 ↕
                          External Network
```

---

### Task 1.1A — Packet Sniffing (ICMP)

#### Approach
Wrote a Python packet sniffer using Scapy to capture ICMP packets on the network interface. Must be run as root to access raw sockets.

```python
#!/usr/bin/env python3
from scapy.all import *

def print_pkt(pkt):
    pkt.show()

pkt = sniff(iface='br-201a717394ac', filter='icmp', prn=print_pkt)
```

Run with:
```bash
sudo python3 sniffer.py
```

Then in another terminal:
```bash
ping 10.9.0.6
```

Successfully captured all ICMP packets between hosts. Running without `sudo` failed — confirming that raw socket access requires root privileges.

#### Key Takeaway
Packet sniffing requires elevated privileges because it uses raw sockets to intercept all traffic on an interface. BPF (Berkeley Packet Filter) syntax is used for the `filter` parameter to select specific traffic types.

---

### Task 1.1B — Packet Sniffing (TCP & Subnet Filters)

#### Approach
Modified the sniffer to capture specific traffic types using different BPF filters.

**TCP filter — capture Telnet traffic from a specific host:**
```python
#!/usr/bin/env python3
from scapy.all import *

def print_pkt(pkt):
    pkt.show()

pkt = sniff(iface='br-201a717394ac',
            filter='tcp and src host 10.9.0.5 and dst port 23',
            prn=print_pkt)
```

**Subnet filter — capture traffic from a specific subnet:**
```python
#!/usr/bin/env python3
from scapy.all import *

def print_pkt(pkt):
    pkt.show()

pkt = sniff(iface='br-201a717394ac',
            filter='net 128.230.0.0/16',
            prn=print_pkt)
```

Both filters successfully captured only the targeted traffic, demonstrating precise traffic selection.

#### Key Takeaway
BPF filters allow sniffers to capture only relevant traffic, reducing noise. Telnet (port 23) transmits credentials in plaintext — sniffing it demonstrates why Telnet should never be used on production systems.

---

### Task 1.2 — ICMP Packet Spoofing

#### Approach
Used Scapy to craft and send a spoofed ICMP packet with a fake source IP address:

```python
#!/usr/bin/env python3
from scapy.all import *

a = IP()
a.src = '1.2.3.4'   # Fake source address
a.dst = '10.9.0.6'  # Real destination

b = ICMP()
p = a/b
send(p)

print(f"Sent spoofed ICMP packet")
```

Run with:
```bash
sudo python3 spoof_icmp.py
```

The destination host `10.9.0.6` received a packet appearing to originate from `1.2.3.4` — a non-existent address — confirming successful spoofing.

#### Key Takeaway
IP source addresses are easily spoofable at the packet level. This is why network security cannot rely on IP addresses alone for authentication. Spoofing is used in DDoS amplification attacks, where responses are directed at the victim rather than the attacker.

---

### Task 1.3 — Custom Traceroute Implementation

#### Approach
Implemented a custom traceroute tool using Scapy by incrementing the TTL (Time To Live) field and recording ICMP Time Exceeded responses:

```python
#!/usr/bin/python3
from scapy.all import *

def tracer(dst, max_hops=30):
    a = IP()
    b = ICMP()
    a.dst = dst

    for ttl in range(1, max_hops + 1):
        a.ttl = ttl
        p = a/b
        reply = sr1(p, timeout=2)
        if reply is None:
            print(f"{ttl}: Request timed out")
        elif reply.type == 0:
            print(f"{ttl}: {reply.src}")
            print("Destination Reached")
            break
        elif reply.type == 11:
            print(f"{ttl}: {reply.src}")
        else:
            print(f"{ttl}: Unexpected reply type: {reply.type}")

tracer('8.8.8.8')
```

The destination `8.8.8.8` was reached with TTL=1 in a single hop. Since the script ran on a host inside the Docker network, it bypassed normal routing — making external destinations appear as single-hop routes.

#### Key Takeaway
Traceroute works by sending packets with incrementing TTL values. Each router decrements the TTL and sends back an ICMP Time Exceeded (type 11) message when TTL reaches 0, revealing its IP. ICMP Echo Reply (type 0) indicates the destination was reached.

---

### Task 2.A — Router Firewall (Protecting the Router)

#### Approach
Configured iptables rules on the router to:
- Allow ICMP (ping) on INPUT and OUTPUT chains
- Block Telnet by setting the default INPUT policy to DROP

```bash
# Allow ICMP
iptables -A INPUT -p icmp -j ACCEPT
iptables -A OUTPUT -p icmp -j ACCEPT

# Block all other incoming connections by default
iptables -P INPUT DROP
```
Ping worked correctly while Telnet connections were blocked.

#### Key Takeaway

iptables processes rules sequentially — the first matching rule wins. Setting the default policy to DROP implements a whitelist approach where only explicitly allowed traffic passes.

---

### Task 2.B — Protecting the Internal Network

#### Approach
Configured firewall rules to allow internal hosts to reach external networks while blocking external hosts from initiating connections to internal hosts:

```bash
# Allow established/related connections back in
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow internal to external
iptables -A FORWARD -s 10.9.0.0/24 -j ACCEPT

# Block external to internal
iptables -A FORWARD -d 10.9.0.0/24 -j DROP
```

#### Key Takeaway
This implements a basic stateful firewall — internal hosts can initiate outbound connections and receive responses, but external hosts cannot initiate inbound connections. This is the standard configuration for most home and enterprise routers.

---

### Task 2.C — Protecting Internal Servers

#### Approach
Configured granular firewall rules to:
- Allow external Telnet access only to `192.168.60.5`
- Block Telnet to all other internal servers
- Allow internal hosts to communicate with each other
- Deny internal hosts from accessing external servers

```bash
# Allow external telnet only to specific server
iptables -A FORWARD -p tcp --dport 23 -d 192.168.60.5 -j ACCEPT

# Block telnet to all other internal hosts
iptables -A FORWARD -p tcp --dport 23 -d 192.168.60.0/24 -j DROP

# Allow internal to internal communication
iptables -A FORWARD -s 192.168.60.0/24 -d 192.168.60.0/24 -j ACCEPT
```

#### Key Takeaway
Granular firewall rules enable precise access control — exposing only specific services on specific hosts rather than entire network segments.

---

### Task 3.A — Connection Tracking Experiment

#### Approach
Used `conntrack` to observe how the router tracks different connection types:

| Protocol | Behavior |
|----------|----------|
| ICMP | Tracked as temporary 30-second flow from last packet |
| UDP (one-way) | Entry expires after 30 seconds |
| UDP (bidirectional) | Entry expires after 180 seconds |
| TCP | Connection tracked while active, removed a few minutes after close |

```bash
conntrack -L
```

#### Key Takeaway
Connection tracking maintains state information about network flows, enabling stateful firewall rules. TCP connections are tracked through their full lifecycle (SYN, ESTABLISHED, FIN), while UDP and ICMP use timeout-based expiry since they have no connection state of their own.

---

### Task 3.B — Stateful Firewall

#### Approach
Configured a stateful firewall using conntrack to automatically permit return traffic for established connections:

```bash
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate NEW -s 10.9.0.0/24 -j ACCEPT
iptables -A FORWARD -j DROP
```

Internal hosts could initiate connections to external hosts and receive responses automatically — without needing explicit rules for return traffic.

#### Key Takeaway
Stateful firewalls track connection state and automatically allow return traffic, eliminating the need for bidirectional rules. This is far more secure than stateless packet filtering which requires explicitly allowing both directions.

---

### Task 4 — Traffic Rate Limiting

#### Approach
Used the iptables `limit` module to restrict traffic from a specific host:

```bash
iptables -A FORWARD -s 10.9.0.5 -m limit --limit 10/minute -j ACCEPT
iptables -A FORWARD -s 10.9.0.5 -j DROP
```

Only a small number of packets per minute from `10.9.0.5` were allowed through — excess packets were dropped.

#### Key Takeaway
Rate limiting is a fundamental DDoS mitigation technique. By capping the number of packets allowed per time unit, it prevents any single host from overwhelming network resources.

---

### Task 5 — Load Balancing

#### Approach
Implemented round-robin load balancing using the iptables `nth` module to distribute traffic across multiple servers:

```bash
iptables -A PREROUTING -t nat -p tcp --dport 80 \
  -m statistic --mode nth --every 2 --packet 0 \
  -j DNAT --to-destination 10.9.0.5:80

iptables -A PREROUTING -t nat -p tcp --dport 80 \
  -j DNAT --to-destination 10.9.0.6:80
```

Round robin was successfully implemented but packets were not split perfectly evenly due to Docker NAT timing — the `nth` module counts packets by arrival time, so any timing deviation throws off the distribution.

#### Key Takeaway
iptables can perform basic load balancing using the `statistic` module. However, production load balancing typically uses dedicated tools like HAProxy or NGINX which handle connection tracking and health checks more reliably.

---

## Lessons Learned

* **Raw socket access requires root** — packet sniffing is a privileged operation by design
* **IP addresses are not authentication** — source IPs are trivially spoofable without network-level controls
* **Stateful firewalls are superior** — tracking connection state eliminates the need for bidirectional rules and is more secure
* **conntrack reveals network behavior** — understanding how TCP, UDP, and ICMP are tracked differently is essential for firewall design
* **Rate limiting is fundamental DDoS mitigation** — simple but effective for preventing resource exhaustion
* **Telnet is insecure** — credentials transmitted in plaintext are visible to any sniffer on the network
