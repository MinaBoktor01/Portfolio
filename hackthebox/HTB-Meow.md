# HackTheBox — Meow (Telnet)
**Platform:** HackTheBox — Starting Point (Tier 0)
**Category:** Network, Telnet
**Difficulty:** Very Easy
## Overview
This challenge involves identifying and exploiting an exposed Telnet service on a Linux target. Telnet transmits data in plaintext and is considered obsolete and insecure. By connecting with the root account and no password, we gained immediate root access and retrieved the flag. 

---
## Attack Chain Summary

### Task 1 — Virtual Machines

#### Challenge Question
In cybersecurity, isolated environments are often VMs. What does VM stand for?
#### Answer
Virtual Machine
#### Key Takeaway
Virtual machines are isolated environments used in both lab setups and real-world attack infrastructure. HTB target machines and tools like Pwnbox are VM-based.

---
### Task 2 — Terminal

#### Challenge Question
What tool do we use to interact with the operating system in order to issue commands via the command line?
#### Answer
Terminal
#### Key Takeaway
The terminal (also called console or shell) is the primary interface for interacting with Linux systems during penetration testing.

---
### Task 3 — VPN Tool

#### Challenge Question
What service do we use to form our VPN connection into HTB labs?
#### Answer
OpenVPN
#### Key Takeaway
HTB uses OpenVPN to tunnel into its private lab network. The connection must be active before interacting with any target machine.

---
### Task 4 — Connection Test

#### Challenge Question
What tool do we use to test our connection to the target with an ICMP echo request?
#### Answer
ping
#### Key Takeaway
Ping is used to verify connectivity to a target before beginning enumeration.

---
### Task 5 — Port Scanner

#### Challenge Question
What is the name of the most common tool for finding open ports on a target?
#### Answer
nmap
#### Key Takeaway
Nmap is the industry standard for port scanning and service enumeration.

---
### Task 6 — Service Identification

#### Challenge Question
What service do we identify on port 23/tcp during our scans?
#### Approach
```bash
nmap -sV 10.129.102.186
```
#### Nmap Output (Relevant)
| Port | State | Service | Version |
|------|-------|---------|---------|
| 23/tcp | open | telnet | Linux telnetd |
#### Answer
telnet
#### Key Takeaway
Port 23 is the default Telnet port. Its presence on a modern system is a major red flag — Telnet sends all data including credentials in plaintext.

---
### Task 7 — Default Credentials

#### Challenge Question
What username is able to log into the target over telnet with a blank password?
#### Answer
root
#### Key Takeaway
Default or unconfigured services often leave the root account accessible with no password. This is one of the most critical misconfigurations possible on a Linux system.

---
### Task 8 — Flag Retrieval

#### Challenge Question
Submit the flag located on the target.
#### Approach
Connected via Telnet using root with no password:

```bash
telnet 10.129.102.186
```

Login: root
Password: (blank)

Once inside, listed and read the flag:

```bash
ls
cat flag.txt
```
#### Flag
b40abdfe23665f766f9c61ecba8a4c19
#### Key Takeaway
Root access with no password over an unencrypted protocol represents a complete system compromise. In a real environment this would expose the entire machine and potentially the network.

---
## Tools Used
| Tool | Purpose |
|------|---------|
| Nmap | Port and service enumeration |
| Telnet | Remote login to target |

## Lessons Learned
- Telnet is old and dangerous ( all traffic including credentials is transmitted in plaintext)
- Root accounts with no password are a critical misconfiguration
- Port 23 open on any modern system should be treated as an immediate red flag
- Always test for default and blank credentials on exposed services
