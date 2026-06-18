# HackTheBox — Dancing (Windows/SMB)
**Platform:** HackTheBox — Starting Point
**Category:** Network, SMB
**Difficulty:** Very Easy

## Overview
This challenge involves enumerating an exposed SMB server on a Windows target. By identifying misconfigured anonymous access on a non-default share, we were able to navigate the file system and retrieve the flag. 

---
## Attack Chain Summary
### Task 1 — SMB Acronym
#### Challenge Question
What does the 3-letter acronym SMB stand for?
#### Answer
Server Message Block
#### Key Takeaway
SMB is a network file sharing protocol used primarily by Windows machines to share files, printers, and communicate over a local network.

---
### Task 2 — SMB Port

#### Challenge Question
What port does SMB use to operate at?
#### Answer
445
#### Key Takeaway
Port 445 is the modern standard for SMB traffic. Legacy SMB also operated over port 139.

---
### Task 3 — Nmap Service Discovery

#### Challenge Question
What is the service name for port 445 that came up in our Nmap scan?
#### Approach
```bash
nmap -sV 10.129.1.12
```
#### Answer
microsoft-ds
#### Key Takeaway
The -sV flag enables service version detection, allowing us to identify what is running on each open port rather than just the port number.

---
### Task 4 — Listing SMB Shares

#### Challenge Question
What is the flag or switch that we can use with the smbclient utility to list the available SMB shares on Dancing?
#### Approach
```bash
smbclient -L 10.129.1.12
```
#### Answer
-L
#### Key Takeaway
The -L flag lists all available shares on a target SMB server. Pressing Enter on the password prompt tests for anonymous/guest access (a common misconfiguration on real targets).

---
### Task 5 — Share Enumeration

#### Challenge Question
How many shares are there on Dancing?
#### Answer
4
#### Shares Discovered
| Share | Type | Notes |
|-------|------|-------|
| ADMIN$ | Disk | Default Windows admin share |
| C$ | Disk | Default Windows drive share |
| IPC$ | IPC | Default interprocess communication share |
| WorkShares | Disk | Non-default — accessible anonymously |

#### Key Takeaway
Default Windows shares (ADMIN$, C$, IPC$) are present on all Windows machines. Non-default shares like WorkShares are prime targets for enumeration.

---
### Task 6 — Accessible Share

#### Challenge Question
What is the name of the share we are able to access in the end with a blank password?
#### Answer
WorkShares
#### Key Takeaway
Anonymous access to non-default shares is a serious misconfiguration that exposes internal files without any authentication.

---
### Task 7 — SMB Download Command

#### Challenge Question
What is the command we can use within the SMB shell to download the files we find?
#### Answer
get
#### Key Takeaway
Inside an SMB shell, get downloads files to the local machine. Standard Linux commands like cat do not work within the SMB shell — files must be downloaded first.

---
### Task 8 — Flag Retrieval

#### Challenge Question
Submit the flag located on the SMB share.
#### Approach
Connected to WorkShares and enumerated two user directories:

```bash
smbclient \\\\10.129.1.12\\WorkShares
```

Navigated Amy.J — found and downloaded worknotes.txt:

```bash
cd Amy.J
get worknotes.txt
```

Contents of worknotes.txt revealed internal task notes:
- Start Apache server on the Linux machine
- Secure the FTP server
- Setup WinRM on Dancing

Navigated James.P — found and retrieved the flag:

```bash
cd James.P
get flag.txt
```
#### Flag
5f61c10dffbc77a704d76016a22f1664

#### Key Takeaway
Sensitive files including flags and internal operational notes were accessible with zero authentication. Real-world equivalents could expose credentials, internal documentation, or PII.

---
## Tools Used
| Tool | Purpose |
|------|---------|
| Nmap | Port and service enumeration |
| smbclient | SMB share enumeration and file retrieval |

## Lessons Learned
- SMB anonymous access is a critical misconfiguration that exposes internal file systems
- Default Windows shares (ADMIN$, C$, IPC$) are distinguishable from custom shares by the $ suffix
- The -sV flag in Nmap identifies running services, not just open ports
- Files must be downloaded with get before reading 
- SMB shell does not support native Linux commands
- Bonus recon in Amy.J revealed the target runs Apache, FTP, and WinRM (useful context for further exploitation)
