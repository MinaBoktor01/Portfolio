## About Me

I'm a Computer Science graduate and Masters student in Cybersecurity at Montclair State University. While my professional experience spans SOC analysis, risk & compliance, and email security, this GitHub is where I document the offensive side of my skillset — CTF writeups, lab walkthroughs, and security research built up through self-study and structured courses.

Everything here is hands-on, documented in my own way of understanding, and performed in legal controlled lab environments.

---

## Certifications

| Certification | Issuer | Year |
|--------------|--------|------|
| Ethical Hacker | Cisco | 2026 |
| CompTIA Security+ | CompTIA | 2025 |
| Google Cybersecurity Professional | Google | 2024 |
| VMDR | Qualys | 2024 |

---

## Writeup Index

### Cisco Networking Academy

| Lab | Topics Covered | Difficulty | Link |
|-----|---------------|------------|------|
| Ethical Hacker CTF | Nmap, Gobuster, Hydra, SQLi, RCE, Privesc | Beginner–Intermediate | [View](./cisco/ethical-hacker-ctf.md) |

---

### TryHackMe

| Room | Topics Covered | Difficulty | Link |
|------|---------------|------------|------|
| Enumeration & Brute Force | Verbose errors, predictable tokens, Basic Auth | Intermediate | [View](./tryhackme/THM-enumeration-bruteforce.md) |
| Sakura Room | OSINT, image metadata, Sherlock, crypto tracing, geolocation | Easy | [View](./tryhackme/THM-Sakura-Room.md) |

---

### Security Research

Hands-on labs covering core offensive and defensive security concepts.

| Lab | Topics Covered | Link |
|-----|---------------|------|
| Buffer Overflow Exploitation | Stack layout, shellcode, NOP sleds, ASLR, StackGuard | [View](./security-research/buffer-overflow-exploitation-lab.md) |
| Network Security | Packet sniffing, ICMP spoofing, traceroute, iptables, stateful firewalls | [View](./security-research/network-security-lab.md) |
| Web Security — CSRF | CSRF GET/POST attacks, token defense, SameSite cookies | [View](./security-research/web-security-csrf-lab.md) |

---

## Attack Methodology

Every writeup follows the standard penetration testing methodology:

```
1. Reconnaissance       →  Identify open ports and services
2. Enumeration          →  Discover hidden paths, users, and data
3. Exploitation         →  Gain initial access
4. Post-Exploitation    →  Establish foothold, hunt for flags
5. Privilege Escalation →  Escalate to root/admin
```

---

## Tools I Use

| Category | Tools |
|----------|-------|
| Recon & Scanning | Nmap, Gobuster, Nikto, Sherlock |
| Exploitation | Hydra, SQLmap, Metasploit, Scapy |
| Shells & Access | Netcat, Python PTY, SSH |
| OSINT | Sherlock, Wigle.net, Etherscan, CyberChef |
| Monitoring & Defense | Splunk, Security Onion, Carbon Black, SentinelOne |
| Infrastructure | pfSense, VMware, VirtualBox, WireGuard |
| Languages | Python, Bash, KQL, Java, C |

---

## Technical Skills

- **Security Tools:** Splunk, Security Onion, Carbon Black, SentinelOne, Trellix Helix, Proofpoint, Qualys, pfSense, Pi-hole, Tailscale
- **Cloud & Infrastructure:** Microsoft Azure, VMware Workstation, VirtualBox, WireGuard VPN, DMARC/DKIM
- **Programming:** Python, Java, C, Bash, KQL
- **OS & Networking:** Windows, Linux (Ubuntu, Arch, Debian, Fedora), VLAN, NAT, UFW, RAID-1
- **Frameworks:** NIST CSF, ISO 27001, Vulnerability Management, Incident Response, Change Management

---

## Notable Projects

**Raspberry Pi Secure Access Point with DNS Filtering** *(May 2025)*
Enterprise-grade wireless access point using Raspberry Pi 4 with Pi-hole DNS filtering, Tailscale VPN, UFW firewall, and NAT — blocking 85% of malicious domains across 50+ users.

**Virtualized Home Lab: Detection & Monitoring** *(July 2024)*
VMware-based cross-platform security lab with Security Onion IDS, pfSense firewall, VLAN segmentation, and real-time monitoring dashboards.

**Home Server with DNS and VPN Access** *(June 2024)*
Ubuntu/CasaOS server with Nextcloud storage, AES-256 encryption, RAID-1 redundancy, WireGuard VPN, and 99.9% uptime.

---

> ⚠️ **Disclaimer:** All activity documented in this repository was performed in legal, controlled lab environments. This portfolio is for educational purposes only.
