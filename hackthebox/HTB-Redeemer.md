# HackTheBox — Redeemer (Redis)
**Platform:** HackTheBox — Starting Point (Tier 0)
**Category:** Database, Redis
**Difficulty:** Very Easy

## Overview
This challenge involves identifying and exploiting an exposed Redis instance on a non-standard port. Redis is an in-memory key-value store that, when left unauthenticated, allows anyone to read all stored data. By connecting with redis-cli and enumerating the keyspace, we retrieved the flag. 

---
## Attack Chain Summary

### Task 1 — Port Discovery

#### Challenge Question
Which TCP port is open on the machine?
#### Approach
Initial default Nmap scan returned no results — Redis runs on a non-standard port outside the top 1000. A full port scan was required:

```bash
nmap -sV -p- 10.129.102.211
```

#### Nmap Output (Relevant)
| Port | State | Service | Version |
|------|-------|---------|---------|
| 6379/tcp | open | redis | Redis key-value store 5.0.7 |

#### Answer
6379
#### Key Takeaway
Always use -p- for a full port scan when default Nmap returns nothing. Services are frequently run on non-standard ports to avoid casual detection. Missing this means missing the target entirely.

---
### Task 2 — Service Identification

#### Challenge Question
Which service is running on the port that is open on the machine?
#### Answer
Redis
#### Key Takeaway
Redis is an open source in-memory key-value store commonly used for caching and session management. When exposed without authentication it is trivially exploitable.

---
### Task 3 — Database Type
#### Challenge Question
What type of database is Redis?
#### Answer
In-memory Database
#### Key Takeaway
Unlike traditional disk-based databases, Redis stores all data in memory making it extremely fast but also meaning all data is immediately accessible when unauthenticated access is possible.

---
### Task 4 — Redis Client

#### Challenge Question
Which command-line utility is used to interact with the Redis server?
#### Answer
redis-cli
#### Key Takeaway
redis-cli is the official Redis command line interface used to connect to and interact with Redis instances.

---
### Task 5 — Hostname Flag

#### Challenge Question
Which flag is used with the Redis command-line utility to specify the hostname?
#### Answer
-h
#### Key Takeaway
The -h flag is consistent across many networking tools for specifying a target hostname or IP address.

---
### Task 6 — Server Info Command

#### Challenge Question
Once connected to a Redis server, which command is used to obtain the information and statistics about the Redis server?
#### Approach
```bash
redis-cli -h 10.129.102.211
```

Then:
```bash
info
```
#### Answer
info
#### Key Takeaway
The info command dumps detailed server statistics including version, OS, memory usage, connected clients, and keyspace data — valuable for enumeration.

---
### Task 7 — Redis Version

#### Challenge Question
What is the version of the Redis server being used on the target machine?
#### Answer
5.0.7
#### Key Takeaway
Identifying the exact version allows us to search for known CVEs and exploits specific to that release.

---
### Task 8 — Database Selection

#### Challenge Question
Which command is used to select the desired database in Redis?
#### Answer
select
#### Key Takeaway
Redis supports multiple databases indexed numerically. Use select followed by the index number to switch between them. Example: select 0

---
### Task 9 — Key Count

#### Challenge Question
How many keys are present inside the database with index 0?
#### Answer
4
#### Key Takeaway
The info command revealed db0:keys=4 under the Keyspace section, showing 4 keys stored in the default database.

---
### Task 10 — List All Keys

#### Challenge Question
Which command is used to obtain all the keys in a database?
#### Answer
keys *
#### Key Takeaway
The * wildcard returns all keys in the current database. In a real engagement this could expose credentials, session tokens, or sensitive application data.

---
### Task 11 — Flag Retrieval

#### Challenge Question
Submit the flag located in the Redis database.
#### Approach
Listed all keys in db0:

```bash
keys *
```

Output:
1) "temp"
2) "numb"
3) "flag"
4) "stor"

Retrieved the flag key:

```bash
get flag
```
#### Flag
03e1d2b376c37ab3f5319922053953eb
#### Key Takeaway
Unauthenticated Redis instances expose all stored data with a single command. In production environments this has led to critical breaches including exposed API keys, session tokens, and user credentials.

---
## Tools Used
| Tool | Purpose |
|------|---------|
| Nmap | Full port scan and service enumeration |
| redis-cli | Redis database interaction and key retrieval |

## Lessons Learned
- Always run nmap -p- when default scans return nothing (services hide on non-standard ports)
- Unauthenticated Redis instances expose all stored data with zero credentials required
- The info command is the first step when enumerating a Redis instance
- Redis keyspace enumeration with keys * can expose credentials, tokens, and sensitive app data in real engagements
- Default Nmap scans only cover the top 1000 ports ( full scans are essential for thorough enumeration)
