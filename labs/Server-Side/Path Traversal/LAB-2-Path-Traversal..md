# Lab 2: File Path Traversal — Traversal Sequences Blocked with Absolute Path Bypass

## Executive Summary

This lab demonstrates a file path traversal vulnerability where the application implements a partial defense — blocking requests containing `../` traversal sequences — but fails to account for an equally effective alternative: supplying an absolute filesystem path directly. When the traversal sequence `../../../etc/passwd` was submitted, the application returned a 400 Bad Request, confirming active filtering of relative traversal patterns. However, by submitting `/etc/passwd` as the filename value with no traversal sequences whatsoever, the application passed the filter and read the file directly from the filesystem, returning its complete contents in the HTTP response. This bypass works because the application's defense is pattern-based rather than path-based — it blocks a specific string pattern while the underlying file read operation accepts any valid path the OS can resolve, including fully qualified absolute paths. The complete contents of `/etc/passwd` were returned, confirming arbitrary filesystem read access via absolute path injection.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | File Path Traversal — Traversal Sequence Filter Bypassed via Absolute Path |
| **Risk**               | Critical - Arbitrary Filesystem Read via Filter Bypass |
| **Completion**         | May 19,2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter where relative traversal sequences (`../`) are blocked by a server-side filter. Bypass the filter by supplying an absolute filesystem path directly, read the contents of `/etc/passwd`, and demonstrate that pattern-based input filtering is an inadequate defense against path traversal.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload testing and response inspection

**Target:** `filename` query parameter in the product image serving endpoint — traversal sequences filtered at the input layer, but underlying file read operation accepts absolute paths without restriction

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, traversal sequences blocked with absolute path bypass" with Burp Suite Professional configured and actively intercepting traffic.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/3f3941f1-6066-44d6-b732-a72d6e1b40d6" />

*Lab 2 Path Traversal environment displaying the traversal sequences blocked with absolute path bypass challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in HTTP history (left)*

### 2. Attack Surface Identification

Browsed the shop product pages with Burp Proxy active to populate HTTP history. Identified the product image loading request as the injection surface — a GET request carrying a `filename` parameter specifying the image file to serve from disk, consistent with Lab 1.

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/8c02be7d-9967-495b-9d13-e6f8651f3557" />

*HTTP history populated with captured requests — image loading request identified and highlighted, showing `filename=24.jpg` as the user-controlled parameter feeding the server-side file read operation*

**Request Captured:**
```http
GET /image?filename=24.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Injection Point:** `filename` parameter — user-supplied value used to construct a filesystem path server-side. Same attack surface as Lab 1, now with an added input filtering layer.

### 3. Baseline Request Validation

Forwarded the image request to Burp Repeater and sent the unmodified request to confirm the file serving mechanism is operational before attempting traversal.

<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/6f37f7be-fc26-44ae-b468-e064bd328f12" />

*Original image request sent in Repeater — 200 OK response returns raw image binary data with Adobe Photoshop metadata visible in the response body, confirming the server reads and returns raw file contents directly from disk*

**Response:** 200 OK — raw image binary data returned, Adobe Photoshop file metadata visible in response.

**Analysis:** Identical baseline behavior to Lab 1. The server reads the file specified by `filename` from a base directory and returns its raw contents. The Photoshop metadata in the response confirms raw file content is returned without transformation, establishing that path traversal to readable files will return their raw contents in the same way.

### 4. Standard Traversal Attempt — Filter Confirmed

Applied the same traversal payload used successfully in Lab 1, substituting the filename value with a relative path traversal sequence targeting `/etc/passwd`.

**Attack Payload:**
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/442684cf-0fa1-4410-800e-728cd5e2fe06" />

*Standard `../../../etc/passwd` traversal payload submitted — 400 Bad Request response confirms the application is actively filtering relative traversal sequences. The `../` pattern is detected and rejected before reaching the file read operation*

**Response:** 400 Bad Request

**Analysis:** The application detects the `../` sequence in the filename parameter and rejects the request before it reaches the filesystem read operation. A filtering mechanism is active — relative path traversal in its standard form is blocked. The defense is pattern-based: it identifies and blocks a specific string pattern (`../`) rather than validating the resolved destination path.

**What This Filter Does and Does Not Cover:**

The filter blocks requests containing `../`. It does not necessarily validate:
- Whether the resolved path falls within the intended base directory
- Whether the supplied value is an absolute path beginning with `/`
- Whether other encoding variants of `../` are present

A pattern-based filter that targets `../` specifically leaves absolute path injection completely unaddressed, because absolute paths contain no `../` sequences — they specify the target location directly without any traversal.

### 5. Absolute Path Bypass — Filter Circumvented

With relative traversal blocked, the next logical approach is to eliminate the traversal sequences entirely. If the application passes the user-supplied filename value directly to a filesystem read operation, and that operation accepts absolute paths, then supplying `/etc/passwd` as the filename value bypasses the filter completely — no `../` sequences are present to trigger it.

**Bypass Payload:**
```http
GET /image?filename=/etc/passwd HTTP/1.1
```

**Why This Works:**

The filter checks for `../` — this payload contains none. The filter passes the request. The server-side file read operation then receives `/etc/passwd` as the filename value. Depending on how the base directory is applied:

```python
# If the server does naive concatenation:
base_dir = "/var/www/images/"
path = base_dir + "/etc/passwd"
# Result: "/var/www/images//etc/passwd"
# On Linux, a leading / in a path component overrides everything before it
# os.path.join("/var/www/images/", "/etc/passwd") → "/etc/passwd"
# The absolute path takes precedence — base directory is discarded
```

On Linux and most Unix-like systems, when a path component begins with `/`, it is treated as an absolute path and the preceding path components are discarded. This is standard POSIX path resolution behavior. The application's base directory concatenation is bypassed by the leading slash in the payload — the OS resolves the path to `/etc/passwd` directly regardless of what base directory the application intended to prepend.

<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/171917da-b72c-4382-ae2a-aad08c92130b" />

**Absolute path `/etc/passwd` submitted as filename value — 200 OK response returns the complete contents of `/etc/passwd` including entries for root, daemon, bin, sys, and all configured system accounts. Filter bypassed, arbitrary filesystem read confirmed*

**Response:** 200 OK — complete contents of `/etc/passwd` returned.

**Response Contents Included:**
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
[all remaining account entries...]
```

**Filter Bypass Confirmed:**
- `../` filter triggered and blocked the standard traversal payload (SS4)
- Absolute path `/etc/passwd` contained no `../` sequences — passed the filter
- Server-side file read resolved `/etc/passwd` as an absolute path, discarding the base directory
- Complete file contents returned in HTTP response

### 6. Lab Objective Achieved

<img width="1920" height="982" alt="lAB2_ss6" src="https://github.com/user-attachments/assets/cb8a9874-3e83-448d-b8f1-5ba3b7c7e643" />

*Lab completion screen — "Congratulations, you solved the lab!" confirming successful bypass of the traversal sequence filter via absolute path injection*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerabilities:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- `filename` parameter value reaches a filesystem read operation without adequate restriction
- Pattern-based filter blocks `../` but does not validate the resolved destination path
- Absolute paths accepted and resolved by the OS, discarding the intended base directory

**CWE-116: Improper Neutralization of Input — Incomplete Filter Implementation**
- Filter addresses one specific attack pattern (`../`) while leaving the equivalent absolute path vector entirely unaddressed
- Defense implemented at the wrong layer — string pattern matching rather than path canonicalization and boundary enforcement
- Incomplete filtering creates a false sense of security while leaving the vulnerability fully exploitable

**Complete Attack Chain:**

```
Image Filename Parameter Identified as Injection Surface
        ↓
Baseline Request Confirms Raw File Contents Served from Disk
        ↓
Standard ../../../etc/passwd Payload Submitted
        ↓
400 Bad Request — ../ Filter Confirmed Active
        ↓
Filter Analysis — Pattern-Based, Does Not Cover Absolute Paths
        ↓
/etc/passwd Submitted as Filename — No ../ Sequences Present
        ↓
Filter Passes Request — No Blocked Pattern Detected
        ↓
OS Resolves /etc/passwd as Absolute Path — Base Directory Discarded
        ↓
Complete /etc/passwd Contents Returned in HTTP Response
        ↓
Arbitrary Filesystem Read Confirmed via Absolute Path Bypass
```

**POSIX Absolute Path Precedence — Why the Base Directory Is Discarded:**

The core behavior enabling this bypass is a fundamental property of POSIX path resolution. When a path component beginning with `/` is encountered, all preceding path components are discarded:

```python
import os

# Standard concatenation — absolute path overrides base directory
os.path.join("/var/www/images/", "/etc/passwd")
# Returns: "/etc/passwd"
# The base directory "/var/www/images/" is completely discarded

# Only a relative path respects the base directory
os.path.join("/var/www/images/", "24.jpg")
# Returns: "/var/www/images/24.jpg"
```

This is not a bug in Python or the OS — it is specified behavior in POSIX. Every major language and runtime follows the same rule: `os.path.join`, Java's `Paths.get()`, PHP's `realpath()`, and Node's `path.join()` all discard preceding components when a leading slash is encountered. An application that naively concatenates a base directory with user input is vulnerable to this regardless of the programming language, unless the concatenation is preceded by path canonicalization and boundary validation.

**Path Traversal Bypass Technique Comparison:**

| Technique | Payload Example | Bypass Condition |
|-----------|----------------|-----------------|
| Standard traversal (Lab 1) | `../../../etc/passwd` | No filter present |
| **Absolute path (this lab)** | `/etc/passwd` | Filter blocks `../` but not leading `/` |
| URL-encoded traversal | `%2e%2e%2f%2e%2e%2fetc/passwd` | Filter checks decoded input only |
| Double URL-encoded | `%252e%252e%252fetc/passwd` | Filter decodes once, misses second encoding |
| Null byte injection | `../../../etc/passwd%00.jpg` | Filter checks extension after null byte |
| Nested traversal | `....//....//etc/passwd` | Filter strips `../` non-recursively |
| OS-specific separators | `..\..\..\etc\passwd` (Windows) | Filter checks `/` only, not `\` |

Each bypass technique exploits a different gap in incomplete filter logic. This lab demonstrates the simplest: a filter targeting a specific string pattern while an equivalent attack vector using different syntax remains completely unaddressed.

**What the Filter Should Have Done:**

The application's filter is implemented at the wrong layer. Checking for `../` in the raw input string is a string operation on unresolved input. The correct approach validates the resolved, canonicalized path after all OS path processing has been applied:

```python
import os

base_dir = "/var/www/images/"
user_input = request.GET["filename"]

# Canonicalize first — resolve all ../, leading /, symlinks
canonical = os.path.realpath(os.path.join(base_dir, user_input))

# Then validate — check the resolved path, not the raw input
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400  # Reject — path escapes permitted directory

return open(canonical, "rb").read()
```

Under this implementation:
- `filename=../../../etc/passwd` → resolves to `/etc/passwd` → fails prefix check → rejected
- `filename=/etc/passwd` → resolves to `/etc/passwd` → fails prefix check → rejected
- `filename=24.jpg` → resolves to `/var/www/images/24.jpg` → passes prefix check → served

Both attack variants are rejected by the same validation logic because it operates on the resolved path, not the raw input string.

**Real-World Impact:**

| Target File | Contents | Consequence |
|-------------|----------|-------------|
| `/etc/passwd` | System accounts (used here) | User enumeration |
| `/etc/shadow` | Password hashes (if server runs as root) | Offline credential cracking |
| `/proc/self/environ` | Process environment variables | Application secrets, API keys, database credentials |
| `~/.ssh/id_rsa` | SSH private key | Direct server access |
| Application config files | Database credentials, secret keys | Full application compromise |
| `/var/log/auth.log` | Authentication events | Internal user activity, credential patterns |

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Attempt standard traversal first to determine whether a filter is present and what it blocks
- Treat a 400 response on traversal as a signal to analyze filter scope, not as a dead end
- Test absolute path injection immediately after relative traversal is blocked — it is the simplest bypass and the first to attempt
- Understand POSIX path resolution behavior to reason about why certain inputs bypass base directory enforcement

**Critical Security Insights:**

**1. Pattern-Based Filters Are Not Path Security:**
Blocking `../` is a string operation — it has no relationship to where the resolved path actually points on the filesystem. A filter that blocks `../` while accepting `/` has removed one attack variant while leaving an equivalent one completely open. String-level filtering of path traversal is inherently incomplete because there are always alternative ways to express the same filesystem destination.

**2. The Fix Must Operate on the Resolved Path, Not the Raw Input:**
Input validation for filesystem paths must occur after canonicalization — after all `../`, leading slashes, symlinks, and relative components have been resolved by the OS. Validating the raw string before resolution is validating something different from what the OS will actually access. The check and the access must agree on what path they are evaluating.

**3. Incomplete Filtering Creates False Security:**
An application that blocks standard traversal but is still fully exploitable via absolute path gives developers and security reviewers a false impression of protection. The filter is visible, it activates, it returns 400 — it appears to be working. The vulnerability remains completely open. Incomplete mitigations that appear functional are in some respects more dangerous than no mitigation at all, because they discourage further investigation.

**4. Indirect File References Eliminate the Attack Surface:**
The most robust fix does not attempt to validate user-supplied filesystem paths — it eliminates them. Mapping user-supplied identifiers to server-controlled filesystem paths removes the ability to influence path resolution entirely:

```python
# No user-supplied path ever reaches the filesystem
IMAGE_PATHS = {
    "24": "/var/www/images/product_24.jpg",
    "25": "/var/www/images/product_25.jpg",
}
path = IMAGE_PATHS.get(request.GET["id"])
if path is None:
    return 404
return open(path, "rb").read()
```

No traversal sequence or absolute path in the `id` parameter produces a valid dictionary key. The user-supplied value never participates in path construction.

**5. Every Path Traversal Filter Should Be Tested for Absolute Path Acceptance:**
In a real engagement, discovering that `../` is blocked should immediately prompt testing of `/etc/passwd` directly. This is the first and simplest bypass to attempt before moving to encoding variants. The pattern is consistent across languages and frameworks — filter implementations that focus on `../` frequently overlook the leading-slash absolute path vector entirely.

## References

1. PortSwigger Web Security Academy - Path Traversal
2. PortSwigger Web Security Academy - Path Traversal Bypass Techniques
3. OWASP Top 10 (2021) - A01: Broken Access Control
4. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
5. CWE-116 - Improper Neutralization of Special Elements
6. POSIX.1-2017 - Path Resolution Specification
7. OWASP Path Traversal Attack Reference

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, filter bypass attempts, or exploitation of path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
