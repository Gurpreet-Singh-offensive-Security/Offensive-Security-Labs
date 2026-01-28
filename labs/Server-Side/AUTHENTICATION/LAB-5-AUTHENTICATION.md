# Lab 5: Username Enumeration via Response Timing

## Executive Summary

This lab demonstrates an advanced username enumeration vulnerability exploiting response timing differences in password verification. The application exhibits longer processing times for valid usernames due to expensive password hashing operations, creating a timing-based side-channel attack vector. Additionally, I identified and bypassed IP-based brute-force protection using the X-Forwarded-For header. By amplifying timing differences through deliberately long passwords, I successfully enumerated valid usernames and conducted targeted password attacks, achieving complete account compromise.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Username Enumeration via Response Timing |
| **Risk**               | High - Authentication Bypass via Timing Attack |
| **Completion**         | January 3, 2026 |

## Objective
Exploit response timing discrepancies to enumerate valid usernames, bypass IP-based brute-force protection, and conduct password attacks to achieve unauthorized account access.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for timing analysis
- Burp Intruder for automated enumeration
- Burp Proxy for traffic interception

**Target:** Login functionality with timing-based information leakage

## Exploitation Walkthrough

### 1. Environment Setup & Initial Reconnaissance

Accessed the lab environment displaying "Username enumeration via response timing" challenge. Configured Burp Suite Professional for comprehensive timing analysis.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/6b986b97-529f-4f38-a609-27918635eef9" />

*Lab 5 environment - Response timing enumeration vulnerability with Burp Suite Professional configured*

### 2. Authentication Request Capture

Submitted test credentials to capture authentication request structure and establish baseline timing measurements.

**Test Credentials:**
- Username: `carlos`
- Password: `donkey12345`
<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/e81d6f73-fcee-46cd-92b8-a2b3e5065c1f" />

*Invalid credentials submitted | POST request captured in Burp Proxy HTTP history showing username=carlos&password=donkey12345*

**Request Structure:**
```http
POST /login HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&password=donkey12345
```

### 3. Invalid Username Timing Analysis

Sent request to Burp Repeater to analyze response timing characteristics for invalid username scenarios.

<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/443db114-4a76-49c3-b2be-98b313e5fe43" />

*Invalid username timing test - Response completed in 206 milliseconds*

**Timing Result:** Invalid username with invalid password completed in **206 ms**.

**Analysis:** Fast response indicates username validation fails early, preventing expensive password hashing operations.

### 4. Valid Username Timing Analysis

Tested with known valid credentials (provided for testing) to compare timing behavior.

**Test Credentials:**
- Username: `wiener` (valid)
- Password: `peter` (valid)
<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/8d9d729f-d4d4-4e28-8177-1fb3d539787d" />

*Valid username timing test - Response completed in 323 milliseconds*

**Timing Result:** Valid username with valid password completed in **323 ms**.

**Critical Discovery:** Valid usernames take significantly longer to process (~117ms difference), indicating:
- Username validation succeeds
- Application proceeds to password verification
- Expensive hashing operation executed (bcrypt, scrypt, etc.)
- Timing difference exploitable for enumeration

**Vulnerability Confirmed:** Response timing correlates directly with username validity, creating a timing-based side-channel attack.

### 5. Brute-Force Protection Discovery

Attempted automated username enumeration via Burp Intruder but encountered rate limiting.

<img width="1920" height="982" alt="LAB2_ss5 1" src="https://github.com/user-attachments/assets/53256ae5-7333-486b-abcd-a5cbff489f9b" />

*Brute-force protection triggered - "You have made too many incorrect login attempts. Please try again in 30 minutes." (highlighted in red)*

**Protection Mechanism Identified:**
- Account lockout after 3 failed attempts
- 30-minute lockout period
- IP-based tracking
- Blocks automated enumeration attempts

**Challenge:** Standard enumeration blocked by rate limiting, requiring bypass technique.

### 6. X-Forwarded-For Header Bypass

**Bypass Hypothesis:** Application trusts X-Forwarded-For header for IP identification, enabling rate limit evasion.

**Test Implementation:** Added `X-Forwarded-For: 1` header to authentication request.
<img width="1920" height="982" alt="LAB2_ss5 2" src="https://github.com/user-attachments/assets/c91a43e2-a8ee-4447-b85e-0441ba2a63ca" />

*X-Forwarded-For header added - Rate limiting successfully bypassed*

**Request with Bypass:**
```http
POST /login HTTP/1.1
Host: [lab-id].web-security-academy.net
X-Forwarded-For: 1
Content-Type: application/x-www-form-urlencoded

username=test&password=test
```

**Bypass Successful:** Application treated request as originating from different IP, resetting rate limit counter.

### 7. Initial Enumeration Attempt

Configured Intruder attack with X-Forwarded-For header rotation to bypass rate limiting.
<img width="1920" height="982" alt="LAB2_ss5 3" src="https://github.com/user-attachments/assets/b6a9c03c-a817-4a52-b704-7bfd742ebe78" />

*Username enumeration with X-Forwarded-For bypass - No blocking observed, but timing differences minimal (highlighted entries in orange)*

**Attack Configuration:**
- X-Forwarded-For: Incremented for each request (1, 2, 3, etc.)
- Target Parameter: Username
- Password: Standard length test password

**Problem Identified:** Timing differences too subtle for reliable detection:
- Response times clustered between 200-350ms
- Natural network variance obscures signal
- Highlighted entries (orange) show minor variations
- No clear differentiation between valid/invalid usernames

**Challenge:** Standard password length insufficient to amplify timing differences for reliable enumeration.

### 8. Timing Amplification Technique

**Solution:** Use extremely long password to amplify timing differences in password verification process.

**Theory:** 
- Invalid username: Fast failure (no password hashing)
- Valid username: Extended processing time due to hashing long password

**Test Implementation:** Modified request in Repeater with artificially long password.

<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/f3f32530-ccd1-4e52-8a5f-dd2dc941e271" />

*Timing amplification test - Password extended with 'peter' + '111' repeated multiple times | Response time: 10,320 milliseconds with valid username*

**Amplified Test:**
```http
POST /login HTTP/1.1
X-Forwarded-For: 12

username=wiener&password=peterpeterpeterpeterpeterpeter111111111111111111111111111111111111111.....
```

**Result:** Response completed in **10,320 ms** with valid username.

**Breakthrough Discovery:** Extremely long passwords force expensive hashing operations, creating massive timing differences:
- Invalid username: ~200ms (immediate rejection)
- Valid username with long password: ~10,000ms (full hashing operation)
- 50x timing amplification achieved

This amplification makes timing differences easily detectable even with network jitter.

### 9. Successful Username Enumeration

Reconfigured Intruder attack with amplified timing technique to enumerate valid usernames.

**Attack Configuration:**
- X-Forwarded-For: Incremented (bypasses rate limiting)
- Target Parameter: Username
- Password: Extremely long string (100+ characters)
- Detection: Response time analysis

<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/c4c87572-c950-4b6a-83d1-1704ea9294c0" />

*Username enumeration successful - Username "accounting" identified with 10,244ms response time (highlighted in green)*

**Attack Results:**

| Username | Response Time | Analysis |
|----------|---------------|----------|
| admin | 215 ms | Invalid (fast rejection) |
| test | 198 ms | Invalid (fast rejection) |
| accounting | 10,244 ms | **Valid (extended hashing)** |
| user | 207 ms | Invalid (fast rejection) |

**Valid Username Identified:** `accounting`

The dramatic timing difference (10,244ms vs ~200ms) provides definitive proof of username validity, eliminating false positives from network variance.

### 10. Targeted Password Attack

With valid username confirmed, launched targeted password brute-force attack.

**Attack Configuration:**
- Username: `accounting` (fixed - confirmed valid)
- Target Parameter: Password
- Payload: Common password wordlist
- Detection: HTTP 302 redirect

<img width="1920" height="982" alt="LAB2_ss9" src="https://github.com/user-attachments/assets/5fc86c8b-bf9e-4abe-838b-cc48bc5424e8" />

*Password brute-force successful - Password "hunter" identified via 302 redirect (highlighted in green)*

**Credentials Discovered:**
- Username: `accounting`
- Password: `hunter`

### 11. Account Takeover Confirmation

Authenticated to victim account using discovered credentials to validate complete compromise.

<img width="1920" height="982" alt="LAB2_ss10" src="https://github.com/user-attachments/assets/9c91478f-4d98-49a3-8dd6-5f3451021488" />

*HTTP history showing successful authentication (left) | Victim account accessed (right) - sensitive information and email modification exposed*

**Complete Account Access Achieved:**
- ✅ Full authentication bypass via timing attack
- ✅ Brute-force protection circumvented
- ✅ Valid username enumerated through amplification
- ✅ Password discovered through targeted attack
- ✅ Complete control over victim account

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: High (CVSS 8.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerabilities:**

**1. CWE-208: Observable Timing Discrepancy**
- Response timing correlates with username validity
- Password hashing operations create measurable delays
- Timing side-channel enables reliable enumeration

**2. CWE-346: Origin Validation Error**
- Application trusts X-Forwarded-For header without validation
- Client-controlled header bypasses IP-based rate limiting
- Brute-force protection completely ineffective

**Vulnerability Chain:**
```
Timing Analysis
        ↓
Valid Username Detection (subtle)
        ↓
Amplification via Long Password
        ↓
Clear Timing Difference (10,000ms vs 200ms)
        ↓
X-Forwarded-For Bypass
        ↓
Rate Limiting Circumvented
        ↓
Automated Enumeration
        ↓
Valid Username: "accounting"
        ↓
Targeted Password Attack
        ↓
Complete Account Takeover
```

**Technical Analysis:**

**Timing Discrepancy Root Cause:**
```python
# VULNERABLE CODE PATTERN
def authenticate(username, password):
    user = database.find_user(username)
    
    if user is None:
        return "Invalid credentials"  # Fast return (~200ms)
    
    # Expensive operation only for valid usernames
    if bcrypt.checkpw(password, user.password_hash):  # ~10,000ms for long passwords
        return redirect('/my-account')
    
    return "Invalid credentials"
```

**Why Timing Attacks Succeed:**
- **Early Rejection:** Invalid usernames fail immediately without hashing
- **Expensive Verification:** Valid usernames trigger costly bcrypt/scrypt operations
- **Amplification:** Long passwords increase hash computation time proportionally
- **Measurable Difference:** 50x timing difference exceeds network variance

**X-Forwarded-For Bypass:**
```python
# VULNERABLE RATE LIMITING
def rate_limit_check(request):
    # Trusts client-provided header
    ip = request.headers.get('X-Forwarded-For', request.remote_addr)
    
    if failed_attempts(ip) > 3:
        return "Too many attempts"
    
    # Attacker controls 'ip' value, bypassing limit
```

**Real-World Impact:**

| Attack Scenario | Consequence |
|-----------------|-------------|
| **Mass Enumeration** | Entire user database mappable despite rate limiting |
| **Targeted Attacks** | High-value accounts (admin, finance) identifiable |
| **Credential Stuffing** | Validated usernames increase breach success rate |
| **Advanced Persistent Threat** | Long-term reconnaissance undetectable |

**Attack Complexity:** Moderate - requires understanding of timing attacks and amplification techniques, but execution is straightforward with proper tooling.

## Key Takeaways

**Penetration Testing Methodology:**
- Advanced timing analysis for side-channel exploitation
- Creative amplification techniques to enhance signal detection
- Bypass testing of security controls (rate limiting)
- Multi-stage attack orchestration (enumeration → bypass → compromise)

**Critical Security Insights:**

**1. Timing Attacks Remain Viable:**
Despite being known for decades, timing-based side channels persist in modern applications. Even subtle timing differences become exploitable with amplification techniques.

**2. Constant-Time Operations Are Mandatory:**
Authentication must use constant-time comparison for all operations:
- Username validation timing must be identical
- Password verification must occur regardless of username validity
- Dummy operations needed for invalid usernames

**3. Client-Controlled Headers Are Untrusted:**
X-Forwarded-For and similar headers must never be trusted for security decisions:
- Attackers fully control HTTP headers
- Rate limiting must use actual connection IP
- Proxy headers require validation and sanitization

**4. Defense-in-Depth Amplification:**
Multiple security weaknesses compound into critical vulnerabilities:
- Timing leak alone: Slow, difficult enumeration
- Rate limit bypass alone: Enables automation
- Combined: Rapid, reliable account enumeration

**5. Amplification Defeats Subtlety:**
Even microsecond timing differences become exploitable when amplified. Attackers can:
- Use extremely long inputs to magnify processing time
- Repeat requests to average out network jitter
- Apply statistical analysis to detect patterns

**Professional Skills Demonstrated:**
- Advanced side-channel attack techniques
- Security control bypass methodologies
- Statistical timing analysis
- Creative problem-solving (amplification technique)
- Comprehensive documentation of sophisticated attacks

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-208 - Observable Timing Discrepancy
4. CWE-346 - Origin Validation Error
5. NIST SP 800-63B - Digital Identity Guidelines
6. Paul C. Kocher - "Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS"

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
