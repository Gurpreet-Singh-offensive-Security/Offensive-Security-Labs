# Lab 5: File Path Traversal — Validation of Start of Path

## Executive Summary

This lab demonstrates exploitation of a file path traversal vulnerability where the application implements a prefix validation defense — verifying that the supplied filename value begins with the expected base directory path `/var/www/images` before passing it to the filesystem read operation. This is a more semantically aware defense than the pattern-based filters in Labs 2, 3, and 4, because it attempts to enforce a location constraint rather than simply blocking specific character sequences. However, the validation checks only the start of the string and does not account for traversal sequences appearing after the required prefix. By supplying a filename that satisfies the prefix requirement and then appends traversal sequences to escape the base directory, the validation passes and the traversal executes. The payload `/var/www/images/../../../../etc/passwd` satisfies the prefix check, traverses four levels up the directory tree, and resolves to `/etc/passwd`. The complete file contents were returned in the HTTP response, confirming arbitrary filesystem read access via start-of-path validation bypass.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | Path Traversal |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | File Path Traversal — Prefix Validation Bypassed via Post-Prefix Traversal |
| **Risk**               | Critical - Arbitrary Filesystem Read via Start-of-Path Bypass |
| **Completion**         | May 23, 2026 |

## Objective

Exploit a file path traversal vulnerability in the product image `filename` parameter where the application validates that the supplied value begins with `/var/www/images` before reading the file. Bypass the prefix validation by supplying a filename that satisfies the required prefix and then appends directory traversal sequences that escape the base directory, demonstrating that start-of-path validation without canonicalization is an inadequate defense.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload testing and response inspection

**Target:** `filename` query parameter in the product image serving endpoint — prefix validation applied to enforce base directory, but traversal sequences appended after the valid prefix are not checked against the resolved path

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "File path traversal, validation of start of path" with Burp Suite Professional configured and actively intercepting traffic, with captured requests visible in the Proxy Intercept panel.

<img width="1920" height="982" alt="LAB5_ss1" src="https://github.com/user-attachments/assets/75b08706-5c89-43f4-9db7-2cb31fe529a9" />

*Lab 5 Path Traversal environment displaying the validation of start of path challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in the Proxy Intercept panel (left)*

### 2. Attack Surface Identification

Browsed the shop product pages with Burp Proxy active to populate HTTP history. Identified the product image loading request as the injection surface. Notably, unlike all preceding labs where the filename value was a simple filename such as `23.jpg`, this request exposes the full filesystem path in the parameter value — `/var/www/images/12.jpg` — which is a direct indication of how the application constructs its file paths and what prefix it expects.


<img width="1920" height="982" alt="LAB5_ss2" src="https://github.com/user-attachments/assets/ba2fff21-75f6-4c71-8f58-06afcd177c9c" />

*HTTP history showing captured image request — `filename=/var/www/images/12.jpg` identified, with the full base directory path visible in the parameter value. This immediately reveals the expected prefix the validation logic enforces and the filesystem location the application considers its base directory*

**Request Captured:**
```http
GET /image?filename=/var/www/images/12.jpg HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Critical Observation:** The filename parameter carries the full absolute path `/var/www/images/12.jpg` rather than just a relative filename. This is a significant disclosure — it reveals:
- The base directory the application uses: `/var/www/images/`
- That the application accepts and expects absolute paths in this parameter
- That a prefix validation check almost certainly requires the value to begin with `/var/www/images`
- The exact prefix that must be present in any bypass payload

**Inferred Server-Side Validation Logic:**
```python
filename = request.GET["filename"]

# Prefix check — enforces expected base directory
if not filename.startswith("/var/www/images"):
    return 400   # Reject — prefix missing

# Passes validation — file read proceeds
return open(filename, "rb").read()
```

The validation checks the start of the string. It does not canonicalize the path or verify that the resolved destination falls within `/var/www/images/` after all traversal sequences are processed.

### 3. Standard Traversal Attempt — Prefix Validation Triggered

Forwarded the request to Burp Repeater and applied the standard relative traversal payload, replacing the filename value entirely.

**Payload:**
```http
GET /image?filename=../../../etc/passwd HTTP/1.1
```
<img width="1920" height="982" alt="LAB5_ss3" src="https://github.com/user-attachments/assets/2a614eba-8f5c-4cfc-88d5-bb65786271ae" />


*Standard `../../../etc/passwd` payload submitted — 400 Bad Request confirms the prefix validation is active. The payload does not begin with `/var/www/images` and is rejected immediately by the start-of-path check*

**Response:** 400 Bad Request

**Analysis:** The payload `../../../etc/passwd` does not begin with `/var/www/images` and is rejected by the prefix check before reaching the filesystem read operation. The validation logic is functioning as designed for this specific input. The bypass requires a payload that satisfies the prefix requirement while still escaping the intended base directory through traversal sequences placed after the valid prefix.

### 4. Prefix-Satisfying Traversal Bypass

The prefix check requires the filename to begin with `/var/www/images`. Satisfying this requirement while still escaping the directory is straightforward: supply the required prefix, then append traversal sequences that navigate up and out of the base directory.

The base directory `/var/www/images` is three levels deep from the filesystem root (`/var`, `/var/www`, `/var/www/images`). Three `../` sequences from inside `/var/www/images/` reach the filesystem root. A fourth is added as a safety margin to account for any trailing directory component, and then `/etc/passwd` is specified as the target.

**Attack Payload:**
```http
GET /image?filename=/var/www/images/../../../../etc/passwd HTTP/1.1
```

**Prefix Validation — Why This Passes:**
```
Supplied value:  /var/www/images/../../../../etc/passwd
Prefix check:    does it start with "/var/www/images"?
                 /var/www/images/../../../../etc/passwd
                 ^^^^^^^^^^^^^^^ YES — prefix present
Validation:      passes
```

The check sees the required prefix at the start of the string and considers the input valid. It does not evaluate what comes after the prefix or where the path resolves after the OS processes the traversal sequences.

**Path Resolution — What the OS Actually Does:**
```
Input to file read:  /var/www/images/../../../../etc/passwd

OS path resolution:
  /var/www/images/  +  ../  →  /var/www/
  /var/www/         +  ../  →  /var/
  /var/             +  ../  →  /
  /                 +  ../  →  /         (cannot go above root)
  /                 +  etc/passwd  →  /etc/passwd

Final resolved path:  /etc/passwd
```

Four traversal sequences from `/var/www/images/` — three to reach the filesystem root, one absorbed at the root level (the OS does not error on `/../` at root, it simply stays at `/`) — followed by `etc/passwd`. The resolved path is `/etc/passwd`, completely outside the intended base directory.

<img width="1920" height="982" alt="LAB5_ss4" src="https://github.com/user-attachments/assets/ae2a418f-e37b-4cc7-a221-1430b8154ca2" />


*Prefix-satisfying traversal payload `/var/www/images/../../../../etc/passwd` submitted — 200 OK response returns the complete contents of `/etc/passwd` including root, daemon, bin, sys, and all configured system account entries. Prefix validation passed; traversal resolved to /etc/passwd*

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
- Prefix `/var/www/images` present at start of value — validation passed
- Traversal sequences `../../../../` appended after prefix — not evaluated by validation
- OS resolved the full path to `/etc/passwd` — base directory escaped
- Complete file contents returned

### 5. Lab Objective Achieved

<img width="1920" height="982" alt="LAB5_ss5" src="https://github.com/user-attachments/assets/e833c271-0b6a-4c8a-9930-c56b2c51ca91" />

*Repeater response panel showing complete `/etc/passwd` contents with all system account credentials exposed (left) | Lab completion confirmation — "Congratulations, you solved the lab!" visible in the browser (right)*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerabilities:**

**CWE-22: Improper Limitation of a Pathname to a Restricted Directory (Path Traversal)**
- `filename` parameter value passed to a filesystem read operation after incomplete validation
- Prefix check confirms the string starts with the expected base directory but does not validate the resolved destination
- Traversal sequences appended after the valid prefix escape the base directory after OS path resolution

**CWE-20: Improper Input Validation — Incomplete Path Validation**
- Validation operates on the raw input string, not the canonicalized resolved path
- String prefix match is not equivalent to filesystem boundary enforcement
- A string starting with `/var/www/images` does not guarantee the resolved path falls within `/var/www/images/`

**Complete Attack Chain:**

```
Image Filename Parameter Identified — Full Path /var/www/images/12.jpg Exposed
        ↓
Base Directory and Prefix Requirement Inferred from Parameter Value
        ↓
Standard ../../../etc/passwd Submitted → 400 (Prefix Missing — Rejected)
        ↓
Prefix Constraint Analyzed — Must Begin with /var/www/images
        ↓
/var/www/images/../../../../etc/passwd Constructed
        ↓
Prefix Check: Starts with /var/www/images — Validation Passes
        ↓
OS Resolves ../../../../ from /var/www/images/ → Filesystem Root
        ↓
/etc/passwd Resolved and Read
        ↓
Complete File Contents Returned — Arbitrary Filesystem Read Confirmed
```

**Why Prefix Validation Is Insufficient:**

Prefix validation on a filesystem path is a string operation that does not reflect filesystem semantics. The string `/var/www/images/../../../../etc/passwd` begins with `/var/www/images` — the prefix check is factually satisfied. But the path the OS resolves from that string is `/etc/passwd` — completely outside `/var/www/images/`. The check and the filesystem disagree on what the input means.

This mismatch is fundamental and not fixable by improving the prefix check. No string-level check on the raw input can reliably determine where the OS will resolve that path, because path resolution involves `../` processing, symlink resolution, and OS-specific separator handling that string operations cannot replicate. The only correct approach is to let the OS resolve the path first and then check the result.

**The Disclosure Problem — Exposing the Base Directory in the Parameter:**

This lab also demonstrates a secondary issue: the application exposes its internal filesystem structure by including the full base directory path in the parameter value. An attacker who sees `filename=/var/www/images/12.jpg` immediately knows:
- The base directory (`/var/www/images/`)
- The filesystem layout (`/var/www/` contains the web root)
- The exact prefix required to bypass start-of-path validation

Applications that expose internal filesystem paths in user-facing parameters are providing attackers with reconnaissance information that directly enables traversal bypass construction. Parameter values should use identifiers or relative filenames, not absolute filesystem paths.

**Comparison of Validation Approaches Across the Path Traversal Series:**

| Lab | Defense Applied | Bypass Technique | Root Failure |
|-----|----------------|-----------------|--------------|
| Lab 1 | None | Direct `../` traversal | No defense |
| Lab 2 | Block `../`, allow `/` | Absolute path `/etc/passwd` | Incomplete pattern coverage |
| Lab 3 | Strip `../` non-recursively | Nested `....//` sequences | Single-pass stripping reconstructs pattern |
| Lab 4 | URL decode then strip | Double URL encoding `%252e%252e%252f` | Decode mismatch between filter and application |
| **Lab 5** | **Prefix must be `/var/www/images`** | **Prefix + traversal `/var/www/images/../../../../etc/passwd`** | **String check does not reflect resolved path** |

Every defense in this series operates on the raw input string and fails because string operations cannot accurately represent filesystem path resolution semantics.

**Real-World Prevalence:**

Start-of-path validation is a commonly implemented but frequently flawed defense. It appears in file download handlers, image serving endpoints, document management systems, and log viewers. The implementation is intuitive — "verify the path starts where I expect it to" — but it conflates string prefix matching with filesystem boundary enforcement. These are not equivalent operations and the difference is exploitable in every case where traversal sequences can appear after the required prefix.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Parameter value analysis before payload construction — the full path `/var/www/images/12.jpg` in the parameter immediately revealed the base directory, the prefix requirement, and the bypass structure
- Standard traversal first to confirm the prefix check is active and understand what it requires
- Bypass payload construction informed by the disclosed base directory — supplying the exact required prefix followed by traversal sequences
- Traversal depth calculation: three sequences to reach root from `/var/www/images/`, one additional for margin

**Critical Security Insights:**

**1. The Full Filesystem Path Should Never Appear in User-Facing Parameters:**
Exposing `/var/www/images/12.jpg` in a URL parameter tells an attacker the exact base directory, the traversal depth needed to reach the root, and the prefix string required to satisfy any start-of-path validation. Applications serving files should use opaque identifiers — integers, UUIDs, or short names — that have no relationship to filesystem paths. The server maintains the mapping; the user never sees a path.

**2. String Prefix Matching Is Not Filesystem Boundary Enforcement:**
A string that starts with `/var/www/images` is not necessarily a path within `/var/www/images/`. The OS resolves path components sequentially, including `../` sequences, symlinks, and mount points. String operations cannot replicate this resolution. Only the OS can determine where a path string actually points, and only after resolution can the result be meaningfully validated against a permitted boundary.

**3. Canonicalization Before Validation Is the Only Correct Pattern:**

**Vulnerable Pattern — Prefix Check on Raw String:**
```python
filename = request.GET["filename"]
if not filename.startswith("/var/www/images"):
    return 400
return open(filename, "rb").read()
```

**Secure Pattern — Canonicalize First, Then Validate:**
```python
import os

base_dir = "/var/www/images/"
filename = request.GET["filename"]

# OS resolves all ../, symlinks, and absolute overrides
canonical = os.path.realpath(os.path.join(base_dir, filename))

# Validate the RESOLVED path — not the raw input string
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400

return open(canonical, "rb").read()
```

Under this implementation, `/var/www/images/../../../../etc/passwd` resolves to `/etc/passwd` before the check runs. The check sees `/etc/passwd`, which does not start with `/var/www/images/`, and correctly rejects the request. The bypass is impossible because the validation operates on the same path representation the filesystem read will use.

**4. Every Path Traversal Defense in This Series Fails for the Same Underlying Reason:**
Labs 1 through 5 demonstrate five different defense implementations. All five fail because they attempt to make security decisions based on the raw input string rather than the canonicalized, resolved path. The canonical fix is identical in all five cases. The lesson is not to implement better string-level defenses — it is to stop using string-level defenses for filesystem path validation entirely.

**5. Indirect File References Remain the Most Robust Solution:**
Eliminating user-controlled filesystem path construction removes the vulnerability class entirely:

```python
# No user input ever reaches path construction
IMAGE_STORE = {
    "12": "/var/www/images/product_12.jpg",
    "13": "/var/www/images/product_13.jpg",
}
image_id = request.GET["id"]
path = IMAGE_STORE.get(image_id)
if path is None:
    return 404
return open(path, "rb").read()
```

No traversal sequence, prefix bypass, encoding variant, or absolute path in `image_id` produces a valid dictionary key. The user-supplied value never participates in path construction and cannot influence what file is read.

## References

1. PortSwigger Web Security Academy - Path Traversal
2. PortSwigger Web Security Academy - Bypassing Path Traversal Defenses
3. OWASP Top 10 (2021) - A01: Broken Access Control
4. CWE-22 - Improper Limitation of a Pathname to a Restricted Directory
5. CWE-20 - Improper Input Validation
6. POSIX.1-2017 - Path Resolution Specification
7. OWASP Path Traversal Attack Reference

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to server filesystems, validation bypass attempts, or exploitation of path traversal vulnerabilities against systems without explicit written authorization is illegal under applicable laws worldwide.
