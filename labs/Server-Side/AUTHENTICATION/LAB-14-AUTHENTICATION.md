# Lab 14: 2FA Bypass Using a Brute-Force Attack

## Executive Summary

This lab demonstrates a critical vulnerability in two-factor authentication implementation where inadequate rate limiting and session management enable MFA code brute-forcing. Despite logout protection after failed attempts, I successfully bypassed these controls using Burp Suite's session handling rules and macros to automate re-authentication and MFA enumeration. By brute-forcing the 4-digit verification code (10,000 combinations) with automated session restoration, I achieved complete account takeover, exposing the severe risks of weak MFA implementations and insufficient attempt limiting.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Insufficient 2FA Rate Limiting & Session Management |
| **Risk**               | Critical - Complete 2FA Bypass via Brute-Force |
| **Completion**         | January 27, 2026 |

## Objective

Bypass two-factor authentication protection by exploiting weak rate limiting and automating session restoration to brute-force MFA codes, achieving unauthorized account access.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Session Handling Rules for automation
- Macro Editor for login flow automation
- Burp Intruder for MFA brute-forcing

**Target:** 2FA implementation with inadequate rate limiting and session management

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed lab environment displaying "2FA bypass using a brute-force attack" challenge and configured Burp Suite Professional.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/46e615bc-5d73-4371-960e-c11f9e8f6d60" />

*Lab 14 environment - 2FA brute-force vulnerability with Burp Suite Professional configured*

### 2. Authentication Flow Analysis

Authenticated with victim credentials (username and password provided) to reach MFA verification stage.

**Known Credentials:**
- Username: `carlos`
- Password: `montoya`
<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/087772b0-3007-459c-b258-9d5d4dbb89c9" />

*MFA code entry page reached (right) | Authentication requests captured in HTTP history (left)*

**Authentication Flow Captured:**
```
Step 1: GET /login
  → Response: Login page with CSRF token

Step 2: POST /login
  → Body: username=carlos&password=montoya&csrf=[TOKEN]
  → Response: 302 redirect to /login2

Step 3: GET /login2
  → Response: MFA code entry page

Step 4: POST /login2
  → Body: mfa-code=[CODE]&csrf=[TOKEN]
  → Response: 302 redirect to /my-account (if code correct)
```

**Request Analysis:**
```http
POST /login2 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

mfa-code=1234&csrf=[CSRF_TOKEN]
```

**Critical Parameters:**
- `mfa-code`: 4-digit verification code (0000-9999)
- `csrf`: Anti-CSRF token
- Session cookie required

**Test Result:** Invalid MFA code returns "Incorrect security code" error.

### 3. Rate Limiting Discovery

Tested MFA submission with multiple incorrect codes to identify brute-force protection.
<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/0853425f-940f-40ec-a243-e20babf65452" />

*Brute-force protection triggered - Account logged out after 2 incorrect MFA attempts*

**Protection Mechanism:**
- **Threshold:** 2 incorrect MFA code submissions
- **Action:** Session terminated, user logged out
- **Impact:** Prevents continuous brute-forcing

**Challenge:** While protection exists, it only terminates the session. If an attacker can automate re-authentication and restore sessions, brute-forcing remains possible.

### 4. Session Handling Rule Configuration

Configured Burp Suite session handling rules to automate login flow and maintain session across MFA attempts.

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/960ea358-c8e1-4bd3-876a-612c277c1b38" />

*Session Handling Rule created with comprehensive scope*

**Rule Configuration:**
- **Rule Name:** Auto-login for MFA bypass
- **Scope:** All tools (Scanner, Repeater, Intruder, Sequencer, Extensions)
- **URL Scope:** Include all URLs
- **Purpose:** Automatically re-authenticate when session expires

### 5. Macro Creation for Login Automation

Created macro to automate the complete authentication flow up to MFA verification.
<img width="1920" height="983" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/bb197ab2-b3ef-49ba-a181-15041d7bc39e" />

*Macro editor showing complete login sequence | Macro test results confirming successful automation*

**Macro Requests Configured:**

**Request 1: GET /login**
- Fetches login page
- Extracts CSRF token
- Response: 200 OK

**Request 2: POST /login**
- Submits username and password
- Uses extracted CSRF token
- Body: `username=carlos&password=montoya&csrf=[TOKEN]`
- Response: 302 Found (redirect to /login2)

**Request 3: GET /login2**
- Accesses MFA entry page
- Establishes authenticated session
- Response: 200 OK

**Macro Test Results:**
- All requests: Successful (200/302 responses)
- CSRF tokens properly extracted and reused
- Session maintained throughout flow
- Ready for automated execution

**Purpose:** This macro ensures that before each MFA attempt, Burp automatically:
1. Obtains fresh CSRF token
2. Re-authenticates with password
3. Reaches MFA page with valid session
4. Enables unlimited MFA brute-forcing

### 6. Session Handling Rule Activation

Verified session handling rule properly configured with macro.
<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/14c52ca2-78f2-49ea-a886-4032b5c5fe3f" />

*Session handling rule active with macro configured for auto-login*

**Configuration Confirmed:**
- ✅ Rule 1 active
- ✅ Macro attached
- ✅ Scope includes all tools
- ✅ Auto-login enabled

### 7. MFA Brute-Force Attack Configuration

Sent MFA submission request to Burp Intruder and configured brute-force attack.

<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/ebe40602-f27d-4e23-96bf-aad07a7eec5d" />

*POST /login2 request sent to Intruder for MFA code enumeration*

**Intruder Configuration:**

**Target Request:**
```http
POST /login2 HTTP/1.1

mfa-code=§0000§&csrf=[TOKEN]
```

**Attack Settings:**
- **Attack Type:** Sniper
- **Payload Position:** `mfa-code` parameter
- **Payload Type:** Numbers
- **Payload Range:** 0000-9999 (all 4-digit combinations)

### 8. MFA Code Brute-Force Execution

**Resource Pool Configuration:**
- **Concurrent Requests:** 1 (single-threaded to avoid rate limiting issues)
- **Purpose:** Sequential testing with session restoration between attempts

**Payload Settings:**
- **Min Value:** 0
- **Max Value:** 9999
- **Step:** 1
- **Format:** 4-digit padded (0000, 0001, 0002, etc.)

<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/dda7816b-5988-4f3c-b55c-1601db3e3e8e" />

*MFA brute-force successful - Valid code 0231 identified (highlighted in green) with session token captured*

**Attack Execution Flow:**

For each MFA code attempt:
1. Macro executes (auto-login)
2. Fresh session established
3. MFA code submitted
4. Response analyzed
5. If failed → Repeat with next code
6. If success → 302 redirect with session token

**Attack Results:**

| MFA Code | Status | Response | Analysis |
|----------|--------|----------|----------|
| 0000 | 200 | Incorrect code | Failed |
| 0001 | 200 | Incorrect code | Failed |
| ... | ... | ... | ... |
| **0231** | **302** | **Redirect to /my-account** | **Success!** |
| ... | ... | ... | ... |

**Valid MFA Code Discovered:** `0231`

**Session Token Captured:**
```http
HTTP/1.1 302 Found
Location: /my-account
Set-Cookie: session=[CARLOS_SESSION_TOKEN]
```

**Attack Success:**
- ✅ 4-digit MFA code brute-forced
- ✅ Session handling macro bypassed logout protection
- ✅ Valid session token obtained
- ✅ Complete authentication achieved

### 9. Session Hijacking & Account Access

Used browser Developer Tools to replace session cookie with captured token from successful MFA verification.

<img width="1920" height="982" alt="LAB2_ss9" src="https://github.com/user-attachments/assets/d1bd8639-8bf0-49d8-bde7-889b47cbc566" />

*Intruder results showing valid MFA and session (left) | Session cookie replaced in browser, victim account accessed (right)*

**Session Hijacking Process:**

**Step 1:** Opened Browser DevTools → Application → Cookies
**Step 2:** Located session cookie
**Step 3:** Replaced value with Carlos's captured token

**Cookie Modification:**
```
Original: session=[CURRENT_SESSION]
Modified: session=[CARLOS_SESSION_TOKEN]
```

**Step 4:** Modified URL from `/login2` to `/my-account`

**Result:** Full access to Carlos's account

**Complete Account Access:**
- ✅ Full 2FA bypass successful
- ✅ Carlos's username and email visible
- ✅ Email modification capability available
- ✅ Complete control over victim account
- ✅ Brute-force protection completely circumvented

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerabilities:**

**1. CWE-307: Improper Restriction of Excessive Authentication Attempts**
- Only 2 MFA attempts allowed per session
- But unlimited sessions can be created
- No account-level attempt tracking
- No time-based lockout mechanism

**2. CWE-308: Use of Single-Factor Authentication**
- MFA implementation provides false security
- Brute-forceable codes (4 digits = 10,000 combinations)
- No rate limiting across sessions
- Protection easily circumvented

**3. CWE-384: Session Fixation**
- Session management allows automated restoration
- No detection of rapid session creation
- No binding between sessions and MFA attempts

**Attack Chain:**
```
Known Credentials (Username + Password)
        ↓
Password Authentication Successful
        ↓
MFA Code Required
        ↓
Session Handling Macro Created
        ↓
Automated Login Flow
        ↓
MFA Brute-Force (0000-9999)
        ↓
Logout After 2 Attempts
        ↓
Macro Re-Authenticates Automatically
        ↓
Continue Brute-Force
        ↓
Valid Code Found: 0231
        ↓
Session Token Captured
        ↓
Complete Account Takeover
```

**Why 4-Digit MFA Codes Are Insufficient:**

| Code Length | Combinations | Brute-Force Time (1 req/sec) | Security Level |
|-------------|--------------|------------------------------|----------------|
| 4 digits | 10,000 | ~2.8 hours | **Weak** |
| 6 digits | 1,000,000 | ~11.6 days | Moderate |
| 8 digits | 100,000,000 | ~3.2 years | Strong |
| TOTP (6 digits, 30s rotation) | 1,000,000 per window | Impractical | **Strong** |

**Real-World Attack Impact:**

| Target | Consequence |
|--------|-------------|
| **Banking** | Complete account access despite 2FA |
| **Email Services** | Mailbox takeover, password resets for other services |
| **Corporate Systems** | Data exfiltration, privilege escalation |
| **Cryptocurrency** | Wallet access, fund theft |
| **Healthcare** | PHI exposure, medical record manipulation |

**Why This Attack is Severe:**

1. **MFA Rendered Useless:** Second factor completely bypassed
2. **Automated Attack:** Session handling enables unattended execution
3. **No Detection:** Appears as legitimate login attempts
4. **Scalability:** Automated against multiple accounts
5. **High Success Rate:** 4-digit codes always crackable

## Key Takeaways

**Penetration Testing Methodology:**
- Session handling rule configuration for complex workflows
- Macro creation for multi-step authentication automation
- Rate limiting bypass through session restoration
- MFA brute-force attack orchestration

**Critical Security Insights:**

**1. MFA Code Length Matters:**
Minimum requirements for security:
- **Never use 4-digit codes** (10,000 combinations = easily brute-forced)
- **Use 6+ digit codes** (1,000,000+ combinations)
- **Prefer TOTP** (time-based, 30-second rotation)
- **Consider push notifications** (eliminates code guessing)

**2. Rate Limiting Must Be Comprehensive:**
Effective MFA protection requires:
- Account-level attempt tracking (not just session-level)
- Global rate limiting across all sessions
- Progressive delays after failures
- Account lockout after threshold
- Time-based restrictions (e.g., 5 attempts per hour)

**3. Session Management During MFA:**
Proper implementation:
- Invalidate all previous sessions on logout
- Detect rapid session creation patterns
- Bind MFA attempts to original authentication session
- Monitor for automated authentication patterns

**4. Defense-in-Depth for 2FA:**
Multiple layers prevent bypass:
- Strong code length (6+ digits or TOTP)
- Comprehensive rate limiting
- Account lockout mechanisms
- IP-based restrictions
- Device fingerprinting
- Behavioral analytics

**5. Automation Detection:**
Applications should detect:
- Rapid sequential MFA attempts
- Multiple session creations from same IP
- Identical user agent patterns
- Suspicious timing between requests

**Professional Skills Demonstrated:**
- Advanced Burp Suite session handling
- Macro creation for complex authentication flows
- Multi-stage attack automation
- Rate limiting bypass techniques
- Understanding of 2FA implementation weaknesses
- Comprehensive attack documentation

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-307 - Improper Restriction of Excessive Authentication Attempts
4. CWE-308 - Use of Single-Factor Authentication
5. NIST SP 800-63B - Digital Identity Guidelines (Multi-Factor Authentication)
6. RFC 6238 - TOTP: Time-Based One-Time Password Algorithm

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
