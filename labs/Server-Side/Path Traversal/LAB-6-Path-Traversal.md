# Lab 6: File Path Traversal — Validation of File Extension with Null Byte Bypass

## Executive Summary

This lab demonstrates exploitation of a file path traversal vulnerability where the application validates that the supplied filename ends with an expected file extension — in this case `.png` — before passing it to the filesystem read operation. This extension-based validation is intended to restrict file access to image files only, preventing traversal to arbitrary system files. Through systematic payload testing, standard traversal, nested sequences, and double URL encoding were all confirmed blocked. The working bypass was achieved by appending a null byte followed by a valid extension — `../../../etc/passwd%00.png` — to the traversal payload. The application's extension check sees `.png` at the end of the string and passes validation. However, when the path reaches the underlying filesystem read operation, the null byte (`%00`, ASCII 0x00) acts as a string terminator in the C-based file handling layer, truncating everything after it. The file system reads `../../../etc/passwd` with the null byte and extension silently discarded. The complete contents of `/etc/passwd` were returned, confirming arbitrary filesystem read access via null byte extension bypass.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | File Path Traversal — Extension Validation Bypassed via Null Byte Truncation |
| **Risk**               | Critical - Arbitrary Filesystem Read via Null Byte Extension Bypass |
| **Completion**         | May 23, 2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter where the application validates that the supplied value ends with an allowed file extension. Bypass the extension check by appending a null byte followed by a valid extension to a traversal payload, causing the validation to pass while the null byte truncates the extension before the filesystem read operation resolves the path, and read the contents of `/etc/passwd`.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for iterative payload testing and response inspection

**Target:** `filename` query parameter in the product image serving endpoint — extension validation applied to enforce allowed file types, but null byte handling in the underlying file read layer allows extension truncation

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, validation of file extension with null byte bypass" with Burp Suite Professional configured and actively intercepting traffic, with captured requests visible in the Proxy Intercept panel.


<img width="1920" height="982" alt="LAB6_ss1" src="https://github.com/user-attachments/assets/7bbf4a34-aeb9-45e2-a83e-21e3315e8715" />

*Lab 6 Path Traversal environment displaying the validation of file extension with null byte bypass challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in the Proxy Intercept panel (left)*

### 2. Attack Surface Identification

Browsed the shop product pages with Burp Proxy active to populate HTTP history. Identified the product image loading request carrying `filename=8.jpg` as the injection surface.


<img width="1920" height="982" alt="LAB6_ss2" src="https://github.com/user-attachments/assets/16a6b500-f428-4a22-94da-f722183d3429" />

*HTTP history showing captured image request — `filename=8.jpg` identified as the user-controlled parameter feeding the server-side file read operation. Extension validation inferred from the challenge title — the application likely checks that the filename ends with an allowed extension before processing*

**Request Captured:**
```http
GET /image?filename=8.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Inferred Server-Side Validation Logic:**
```python
filename = request.GET["filename"]

# Extension check — enforces allowed file types
allowed_extensions = [".jpg", ".jpeg", ".png", ".gif"]
if not any(filename.endswith(ext) for ext in allowed_extensions):
    return 400   # Reject — extension not permitted

# Passes validation — file read proceeds
return open(BASE_DIR + filename, "rb").read()
```

The validation confirms the filename ends with an approved extension. It does not validate the path structure, check for traversal sequences, or canonicalize the resolved path. The bypass requires a payload that ends with a valid extension at the string level while causing the extension to be ignored at the filesystem level.

### 3. Baseline Request Validation

Forwarded the image request to Burp Repeater and sent the unmodified request to confirm the baseline file serving behavior.

<img width="1920" height="982" alt="LAB6_ss3" src="https://github.com/user-attachments/assets/af4e589c-df32-4e8c-8cc2-fa548a8bc09c" />

*Original image request `filename=8.jpg` sent in Repeater — 200 OK response with raw image binary data and Adobe Photoshop metadata visible in the response body, confirming the server reads and returns raw file contents directly from disk. Injection surface and file serving mechanism confirmed*

**Response:** 200 OK — raw image binary data with Adobe Photoshop metadata present.

**Analysis:** Consistent baseline with all preceding path traversal labs. Raw file contents served directly from disk without transformation. Extension validation is the defense to bypass — systematic payload testing begins with known techniques before reaching null byte injection.

### 4. Standard Traversal Attempt — Blocked

Applied the standard relative traversal payload to establish what the filter blocks.

**Payload:**
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```

<img width="1920" height="982" alt="LAB6_ss4" src="https://github.com/user-attachments/assets/cc57e195-21ee-4ae5-a9f9-ebbf6fabf2ba" />

*Standard `../../../etc/passwd` submitted — 400 Bad Request. The payload does not end with an allowed extension — extension validation rejects it immediately*

**Response:** 400 Bad Request

**Analysis:** The payload fails the extension check — `passwd` is not an allowed extension. Extension validation is confirmed active. Moving to alternative bypass techniques.

### 5. Nested Traversal Attempt — Blocked

Tested the nested traversal bypass that succeeded in Lab 3.

**Payload:**
```http
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
```

<img width="1920" height="982" alt="LAB6_ss5" src="https://github.com/user-attachments/assets/40f3c19e-bc8b-4d5c-9c20-f3e2d7ccaf79" />


*Nested traversal `....//....//....//etc/passwd` submitted — 400 Bad Request. Extension check still active — `passwd` does not satisfy the allowed extension requirement regardless of the traversal form used*

**Response:** 400 Bad Request

**Analysis:** The nested sequence bypass from Lab 3 fails here for the same reason as the standard traversal — the payload does not end with a valid extension. The filter is checking the end of the string, not the beginning or middle. Any payload that targets `/etc/passwd` without a valid extension suffix will fail this check. The bypass must append a valid extension to the payload.

### 6. Double URL Encoding Attempt — Blocked

Tested the double URL encoding bypass that succeeded in Lab 4.

**Payload:**
```http
GET /image?filename=%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd HTTP/1.1
```


<img width="1920" height="982" alt="LAB6_ss6" src="https://github.com/user-attachments/assets/58e258cc-d92e-4615-8a0d-291a0f261611" />

*Double URL-encoded traversal `%252e%252e%252f` multiple times with `/etc/passwd` submitted — 400 Bad Request. The extension check sees no valid extension at the end of the decoded string regardless of the encoding applied to the traversal sequences*

**Response:** 400 Bad Request

**Analysis:** Double URL encoding does not help here because the extension validation checks the end of the string — `passwd` is still `passwd` regardless of how the preceding traversal sequences are encoded. The fundamental problem is that `/etc/passwd` does not end with `.jpg`, `.png`, or any other allowed extension. The solution is not to change the traversal encoding but to append a valid extension to the payload while preventing that extension from being included in the actual filesystem path resolution.

**Filter Behavior Summary After Three Failed Attempts:**

| Payload | Response | Inference |
|---------|----------|-----------|
| `../../../etc/passwd` | 400 | Extension check active — `passwd` rejected |
| `....//....//....//etc/passwd` | 400 | Extension check catches all traversal forms ending in `passwd` |
| `%252e%252e%252fetc/passwd` | 400 | Encoding does not change extension at end of string |
| Required bypass | Must end with valid extension AND reach `/etc/passwd` | Null byte truncation |

### 7. Null Byte Extension Bypass — Filter Circumvented

The null byte (`%00`, URL-encoded form of ASCII 0x00) is a string terminator in C and C-derived languages. Many server-side file handling functions — particularly those built on C standard library calls such as `fopen()` — treat the null byte as the end of the string, discarding everything after it. If the application's extension validation operates at a higher language level (Python, Java, PHP string handling) while the file read operation ultimately calls a C-level function, the null byte creates a split: the high-level validation sees the full string including the extension after the null byte, while the C-level file open call sees only the portion before it.

**Attack Payload:**
```http
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
```

**How the Null Byte Splits Validation from Execution:**

```
Full payload string:    ../../../etc/passwd%00.png

Extension validation (high-level string check):
  Sees full string:     ../../../etc/passwd%00.png
  Checks end of string: ends with .png
  Validation result:    PASS — .png is an allowed extension

File read operation (C-level fopen or equivalent):
  Receives string:      ../../../etc/passwd\x00.png
  Null byte encountered at position after "passwd"
  String terminated:    ../../../etc/passwd
  .png discarded:       everything after \x00 is ignored
  File opened:          ../../../etc/passwd → resolves to /etc/passwd
```

The extension check and the file read operation see fundamentally different things. The check sees `.png` at the end and passes. The file system never sees `.png` — it was silently truncated by the null byte before the path reached the OS.


<img width="1920" height="982" alt="LAB6_ss7" src="https://github.com/user-attachments/assets/abed9067-93c4-4737-be9e-7d47bf71e013" />

*Null byte bypass payload `../../../etc/passwd%00.png` submitted — 200 OK response returns the complete contents of `/etc/passwd` including root, daemon, bin, sys, and all configured system account entries. Extension validation passed on `.png`; null byte truncated the extension before filesystem resolution*

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
- Extension check passed — `.png` present at end of string at validation layer
- Null byte `%00` truncated the string at C-level file handling — `.png` discarded
- Traversal payload `../../../etc/passwd` reached the filesystem read operation intact
- OS resolved the path to `/etc/passwd` and returned complete file contents

**Lab objective achieved and confirmed in browser.**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerabilities:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- `filename` parameter value reaches a filesystem read operation without adequate path restriction
- Extension validation provides no traversal protection — it only checks the string end
- Null byte truncation removes the extension before the path reaches the OS, bypassing the only active defense

**CWE-158: Improper Neutralization of Null Byte or NUL Character**
- Null byte in user-supplied input not sanitized before reaching C-level string handling
- Application validation layer and filesystem layer treat null bytes differently
- Null byte acts as a string terminator at the C level, silently discarding all data after it

**CWE-20: Improper Input Validation — Layer Mismatch**
- Security control (extension check) and sensitive operation (file read) operate on different representations of the same string
- High-level language string sees the full value including post-null content
- C-level file function sees only the pre-null content
- The mismatch between these views is the exploitable gap

**Complete Attack Chain:**

```
Image Filename Parameter Identified as Injection Surface
        ↓
Baseline Confirms Raw File Contents Served from Disk
        ↓
../../../etc/passwd → 400 (No Valid Extension)
        ↓
....//....//etc/passwd → 400 (No Valid Extension)
        ↓
%252e%252e%252fetc/passwd → 400 (No Valid Extension)
        ↓
Extension Validation Identified as Sole Active Defense
        ↓
Null Byte Bypass Constructed: ../../../etc/passwd%00.png
        ↓
Extension Check Sees .png at String End — Validation Passes
        ↓
C-Level File Function Encounters \x00 — String Terminated
        ↓
.png Discarded — ../../../etc/passwd Reaches Filesystem
        ↓
/etc/passwd Resolved and Read — Complete Contents Returned
```

**Null Byte Behavior Across Language and Runtime Boundaries:**

The null byte vulnerability exists specifically at the boundary between high-level language string handling and C-level system calls. The behavior differs by layer:

| Layer | Null Byte Handling | Effect on Payload |
|-------|--------------------|-------------------|
| Python string | Null byte is a valid character — string continues | Full string including `.png` seen |
| Java String | Null byte is a valid character — string continues | Full string including `.png` seen |
| PHP string (older versions) | Null byte terminates in some filesystem calls | Extension truncated |
| C `fopen()` / `open()` | Null byte terminates the string | Extension discarded |
| Linux kernel path resolution | Null byte is invalid in paths | Returns error or truncates |

The vulnerability requires a specific combination: high-level validation that does not strip null bytes, combined with a lower-level file operation that terminates on null bytes. Modern PHP versions and many contemporary frameworks sanitize null bytes before they reach filesystem calls, making this technique less universally applicable than it was historically. However, legacy systems, C-based web servers, and applications that pass unsanitized input to C library functions remain susceptible.

**Extension Validation as a Defense — Fundamental Limitations:**

Extension validation addresses a different threat than path traversal — it is designed to prevent access to non-image file types, not to prevent directory traversal. Even a correctly implemented extension check that cannot be bypassed provides no traversal protection by itself:

```
A valid bypass without null byte (if .passwd files existed):
  ../../../etc/shadow.jpg   — passes extension check, still a traversal

A non-traversal bypass:
  malicious_script.php      — fails extension check correctly
```

Extension validation and path traversal protection are separate concerns requiring separate controls. A single defense addressing only one of these concerns leaves the other fully open.

**Complete Path Traversal Defense Failures — Full Series Summary:**

| Lab | Defense | Bypass | Core Failure |
|-----|---------|--------|--------------|
| 1 | None | Direct `../` | No defense |
| 2 | Block `../` | Absolute path `/etc/passwd` | Incomplete pattern coverage |
| 3 | Strip `../` once | Nested `....//` reconstruction | Non-recursive stripping |
| 4 | URL decode then strip | Double encoding `%252e%252e%252f` | Single-pass decode mismatch |
| 5 | Prefix must be `/var/www/images` | Prefix + traversal | String prefix vs resolved path |
| **6** | **Extension must be `.png`/`.jpg`** | **Null byte `%00.png`** | **Layer mismatch — high-level vs C-level string** |

Every defense in this series fails because it operates on the raw input string at a single layer and does not account for how that string is interpreted at a different layer of the processing stack.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Systematic bypass enumeration before reaching null byte: standard traversal, nested sequences, double encoding — each confirms a different aspect of filter behavior and eliminates techniques before selecting the correct one
- Extension validation identified as the sole active defense after three blocked attempts, all for the same reason: no valid extension at end of payload
- Null byte bypass selected based on the specific defense type — null byte is the canonical technique for extension validation bypass
- The bypass logic is informed by understanding the processing stack: high-level validation + C-level file read = null byte truncation opportunity

**Critical Security Insights:**

**1. Extension Validation Does Not Protect Against Path Traversal:**
These are separate vulnerability classes requiring separate controls. An extension check prevents non-image file types from being requested by filename, but it does not prevent the path from escaping the intended directory. Both path canonicalization and extension validation are required independently — one does not substitute for the other.

**2. Null Bytes Must Be Sanitized Before Any String Reaches a File Operation:**
User-supplied input containing null bytes should be rejected or stripped at the earliest possible processing point — before any validation logic runs, not after. If the application strips null bytes before validation, the null byte bypass is eliminated. If it strips them after validation but before the file read, the bypass may still work depending on implementation details. The only safe approach is rejection at input boundary.

**3. Security Controls and Sensitive Operations Must Agree on String Representation:**
The null byte bypass works because the extension check and the file read operation interpret the same string differently. Every security control protecting a sensitive operation must handle the input identically to how that operation will handle it — including null byte behavior, encoding, Unicode normalization, and whitespace handling. Discrepancies between how the control and the operation see the input are exploitable.

**4. The Canonical Fix Remains the Same Across All Six Labs:**

```python
import os

base_dir = "/var/www/images/"
raw_input = request.GET["filename"]

# Step 1 — Reject null bytes before any processing
if '\x00' in raw_input:
    return 400

# Step 2 — Canonicalize the resolved path
canonical = os.path.realpath(os.path.join(base_dir, raw_input))

# Step 3 — Validate against the resolved path boundary
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400

# Step 4 — Optional: validate extension on the canonical path
if not canonical.endswith(('.jpg', '.jpeg', '.png', '.gif')):
    return 400

return open(canonical, "rb").read()
```

Applied in this order: null byte rejection eliminates the truncation attack, canonicalization resolves all traversal sequences, boundary validation catches escaped paths, extension check on the canonical path (not the raw input) validates the file type against the actual resolved filename.

**5. Historical vs. Modern Applicability:**
Null byte injection was a widespread and highly reliable technique against PHP applications prior to PHP 5.3.4, where `fopen()` and related functions passed null bytes directly to C library calls. Since PHP 5.3.4, PHP raises an error when null bytes are encountered in filesystem function arguments. However, the technique remains applicable against legacy PHP deployments, C and C++ web applications, applications using custom C extensions, and any system where user input reaches a C-level `open()` or `fopen()` call without null byte sanitization. It should be included in any path traversal assessment regardless of the apparent technology stack.

## References

1. PortSwigger Web Security Academy - Path Traversal
2. PortSwigger Web Security Academy - Bypassing Path Traversal Defenses
3. OWASP Top 10 (2021) - A01: Broken Access Control
4. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
5. CWE-158 - Improper Neutralization of Null Byte or NUL Character
6. CWE-20 - Improper Input Validation
7. PHP Security Advisory - Null Byte Handling in Filesystem Functions (pre-5.3.4)
8. OWASP Path Traversal Attack Reference
9. OWASP Null Byte Injection Reference

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, null byte injection, or exploitation of path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
