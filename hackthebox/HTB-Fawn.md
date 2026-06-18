# HackTheBox — Fawn (FTP)
**Platform:** HackTheBox — Starting Point (Tier 0)
**Category:** Network, FTP
**Difficulty:** Very Easy

## Overview
This challenge involves identifying and exploiting an exposed FTP service on a Linux target. FTP transmits data in plaintext and supports anonymous login by default if misconfigured. By connecting anonymously we gained access to the file system and retrieved the flag.

---
## Attack Chain Summary
### Task 1 — FTP Acronym

#### Challenge Question
What does the 3-letter acronym FTP stand for?
#### Answer
File Transfer Protocol
#### Key Takeaway
FTP is a standard network protocol used to transfer files between a client and server over a TCP network.

---
### Task 2 — FTP Port

#### Challenge Question
Which port does the FTP service listen on usually?
#### Answer
21
#### Key Takeaway
Port 21 is the default FTP control port. Port 20 is used for active mode data transfer.

---
### Task 3 — Secure Alternative

#### Challenge Question
FTP sends data in the clear, without any encryption. What acronym is used for a later protocol designed to provide similar functionality to FTP but securely, as an extension of the SSH protocol?
#### Answer
SFTP
#### Key Takeaway
SFTP (SSH File Transfer Protocol) is the secure replacement for FTP. Unlike FTP, SFTP encrypts both commands and data in transit.

---
### Task 4 — Connection Test
#### Challenge Question
What is the command we can use to send an ICMP echo request to test our connection to the target?
#### Answer
ping
#### Key Takeaway
Always verify connectivity to the target before beginning enumeration.

---
### Task 5 — Service Version

#### Challenge Question
From your scans, what version is FTP running on the target?
#### Approach
```bash
nmap -sV 10.129.102.193
```

#### Nmap Output (Relevant)
| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | open | ftp | vsftpd 3.0.3 |
#### Answer
vsftpd 3.0.3
#### Key Takeaway
Identifying the exact service version allows us to search for known vulnerabilities specific to that version.

---
### Task 6 — Operating System
#### Challenge Question
From your scans, what OS type is running on the target?
#### Answer
Unix
#### Key Takeaway
Nmap's service detection can reveal the underlying OS, helping tailor our attack approach.

---
### Task 7 — Help Menu

#### Challenge Question
What is the command we need to run in order to display the ftp client help menu?
#### Answer
ftp -?
#### Key Takeaway
Most command line tools support a help flag to display available options and usage syntax.

---
### Task 8 — Anonymous Login

#### Challenge Question
What is the username that is used over FTP when you want to log in without having an account?
#### Answer
anonymous
#### Key Takeaway
Anonymous FTP access is a common misconfiguration that allows anyone to connect without credentials. It is intended for public file distribution but is dangerous when left enabled on sensitive servers.

---
### Task 9 — Login Response Code

#### Challenge Question
What is the response code we get for the FTP message Login successful?
#### Answer
230
#### Key Takeaway
FTP uses numeric response codes to communicate status. 230 means login successful. Knowing these codes helps interpret FTP session output during engagements.

---
### Task 10 — Listing Files

#### Challenge Question
There are a couple of commands we can use to list the files and directories available on the FTP server. One is dir. What is the other that is a common way to list files on a Linux system?
#### Answer
ls
#### Key Takeaway
The ls command works inside the FTP shell to list available files and directories on the remote server.

---
### Task 11 — Download Command

#### Challenge Question
What is the command used to download the file we found on the FTP server?
#### Answer
get
#### Key Takeaway
The get command downloads a file from the remote FTP server to the local machine, identical to SMB shell behavior.

---
### Task 12 — Flag Retrieval

#### Challenge Question
Submit the flag located on the FTP server.
#### Approach
Connected anonymously to the FTP server:

```bash
ftp 10.129.102.193
```

Login: anonymous
Password: (blank)

Listed files and downloaded the flag:

```bash
ls
get flag.txt
```

Read the flag on the local machine:

```bash
cat flag.txt
```
#### Flag
035db21c881520061c53e0536e44f815
#### Key Takeaway
Anonymous FTP access with no password on a production system represents a critical misconfiguration. Combined with sensitive files being stored in the FTP root, this results in full data exposure with zero authentication required.

---
## Tools Used
| Tool | Purpose |
|------|---------|
| Nmap | Port and service enumeration |
| FTP client | Anonymous login and file retrieval |

## Lessons Learned
- Anonymous FTP access is a critical misconfiguration that requires no credentials to exploit
- FTP transmits all data including credentials in plaintext ( SFTP should always be used instead)
- FTP response code 230 confirms successful login
- The get command is consistent across both FTP and SMB shells for file retrieval
- Always check for default and anonymous access on file transfer services during enumeration
