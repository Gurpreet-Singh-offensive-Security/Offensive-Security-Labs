# Lab 12: Password Brute-Force via Password Change

## Executive Summary

This lab demonstrates critical vulnerabilities in password change functionality where error message inconsistencies and broken logout logic enable password enumeration and brute-force attacks. By systematically analyzing password change behavior, I identified that the application reveals whether the current password is correct through distinct error messages when new passwords don't match. Additionally, the application fails to enforce session termination when incorrect passwords are submitted with mismatched new passwords, bypassing brute-force protection. These flaws enabled successful password discovery through automated enumeration, achieving complete account takeover without triggering lockout mechanisms.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Password Enumeration via Password Change Logic Flaws |
| **Risk**               | High - Authentication Bypass & Account Takeover |
| **Completion**         | January 19, 2026 |

## Objective

Exploit password change functionality logic flaws to enumerate victim passwords through error message analysis and broken logout protection, achieving unauthorized account access via brute-force attacks.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Intruder for password enumeration
- Burp Repeater for behavior testing
- Grep Match for response pattern detection

**Target:** Password change functionality with information disclosure and broken access control

## Exploitation Walkthrough

### 1. Environment Configuration & Initial Access

Accessed the lab environment displaying "Password brute-force via password change" challenge. Configured Burp Suite Professional for comprehensive request analysis.
<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/515af8dd-4cfb-409b-8bff-0e2e61717bf4" />

*Lab 12 environment - Password change vulnerability with Burp Suite Professional configured*

### 2. Account Access & Functionality Discovery

Authenticated with my credentials to explore available functionality and identify attack surfaces.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/7552f1f1-cc71-4c38-b5ec-41648999a0af" />

*Account dashboard showing sensitive functionality (right) | Authentication requests captured in Burp HTTP history (left)*

**Sensitive Functionality Identified:**
- Email update capability
- **Password change functionality** ← Primary attack target
- Session management

**HTTP Requests Captured:**
```http
POST /login HTTP/1.1
username=wiener&password=peter

GET /my-account HTTP/1.1
Cookie: session=[SESSION_TOKEN]
```

### 3. Legitimate Password Change Testing

Tested password change functionality with valid inputs to understand normal behavior and request structure.

**Test Parameters:**
- Current Password: `peter` (correct)
- New Password 1: `peter123`
- New Password 2: `peter123` (matching)
<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/c5a66904-5555-41ab-a3cf-4de5395d342d" />

*Successful password change request captured (left) | Confirmation displayed on account page (right)*

**Request Captured:**
```http
POST /my-account/change-password HTTP/1.1
Cookie: session=[SESSION_TOKEN]

username=wiener&current-password=peter&new-password-1=peter123&new-password-2=peter123
```

**Response:** Password changed successfully

**Baseline Established:** Legitimate password changes work correctly with matching new passwords and correct current password.

### 4. Brute-Force Protection Analysis

Systematically tested password change behavior with incorrect current passwords to identify security controls.

**Test Scenario 1: First Incorrect Attempt**
- Current Password: `wrong1` (incorrect)
- New Password 1: `newpass`
- New Password 2: `newpass`

**Result:** Redirected to login page

**Test Scenario 2: Second Incorrect Attempt**
- Current Password: `wrong2` (incorrect)
- New Password 1: `newpass`
- New Password 2: `newpass`

**Result:** Account locked for 1 minute
<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/d841200c-1518-448d-8aa2-975bd84b6209" />

*Brute-force protection triggered - Account lockout after 2 incorrect password attempts*

**Protection Mechanism Identified:**
- **Threshold:** 2 incorrect current password attempts
- **Lockout Duration:** 1 minute
- **Trigger:** Incorrect current password submissions

**Security Assessment:** Application implements basic brute-force protection, but further testing needed to identify bypass opportunities.

### 5. Error Message Analysis - Correct Password Scenario

Tested password change with correct current password but mismatched new passwords to analyze error messaging.

**Test Parameters:**
- Current Password: `peter` (correct)
- New Password 1: `newpass1`
- New Password 2: `newpass2` (different)
<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/93d127a9-0c65-4d53-ae1f-a9791545ccbf" />

*Error message analysis - "New passwords do not match" displayed (right) | POST request captured (left)*

**Request:**
```http
POST /my-account/change-password HTTP/1.1

username=wiener&current-password=peter&new-password-1=newpass1&new-password-2=newpass2
```

**Response:** "New passwords do not match"

**Critical Discovery:**

**Error Message Reveals Password Validity:**
- When current password is **correct** → "New passwords do not match"
- This message confirms the current password is valid
- Application validates current password before checking new password matching

**Information Leakage:** The error message indirectly confirms password correctness, enabling enumeration attacks.

### 6. Error Message Analysis - Incorrect Password Scenario

Tested password change with incorrect current password and mismatched new passwords to compare error behavior.

**Test Parameters:**
- Current Password: `wrongpassword` (incorrect)
- New Password 1: `newpass1`
- New Password 2: `newpass2` (different)

<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/cbba07de-3bb2-4589-953d-8a925a4799ab" />

<img width="1920" height="982" alt="=LAB2_ss6" src="https://github.com/user-attachments/assets/9351a5f8-73d4-4557-8889-6d221e560670" />


*Error message with incorrect current password - "Current password is incorrect" displayed (right) | Request captured (left)*
*Vulnerability confirmation - Mismatched new passwords prevent logout, bypassing brute-force protection*

**Response:** "Current password is incorrect"

**Critical Vulnerability Confirmed:**

**Distinct Error Messages Enable Enumeration:**

| Scenario | Current Password | New Passwords | Error Message | Information Leak |
|----------|------------------|---------------|---------------|------------------|
| 1 | Correct | Matching | Success | Password valid |
| 2 | Correct | Mismatched | "New passwords do not match" | **Password valid** |
| 3 | Incorrect | Mismatched | "Current password is incorrect" | Password invalid |

**Exploitation Vector Identified:**

By submitting mismatched new passwords:
- **Correct current password** → "New passwords do not match"
- **Incorrect current password** → "Current password is incorrect"

Attackers can enumerate passwords by testing candidates with mismatched new passwords and analyzing error messages.


**Second Critical Vulnerability:**

**Broken Logout Protection:**
- Incorrect current password + matching new passwords → Logout triggered
- Incorrect current password + **mismatched new passwords** → **No logout**
- Bypasses account lockout mechanism

**Combined Vulnerability Impact:**

1. **Information Disclosure:** Error messages reveal password validity
2. **Broken Access Control:** Mismatched new passwords prevent logout
3. **Brute-Force Protection Bypass:** Enumeration possible without triggering lockout

### 7. Automated Password Enumeration Attack

Configured Burp Intruder to automate password enumeration exploiting the discovered logic flaws.

**Attack Configuration:**

Sent password change request to Intruder and configured enumeration attack.

<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/548dfbb6-352e-46fa-9821-21c4ddd249a7" />


*Intruder attack configured - Victim password discovered through error message pattern (highlighted in green)*

**Intruder Setup:**

**Target Request:**
```http
POST /my-account/change-password HTTP/1.1

username=carlos&current-password=§payload§&new-password-1=newpass1&new-password-2=newpass2
```

**Attack Parameters:**
- **Username:** `carlos` (victim - fixed)
- **Current Password:** Payload position (password candidates)
- **New Password 1:** `newpass1` (fixed - arbitrary)
- **New Password 2:** `newpass2` (fixed - different from password 1)

**Key Configuration:**
- New passwords intentionally mismatched to:
  1. Generate distinguishable error messages
  2. Prevent logout on incorrect attempts
  3. Bypass brute-force protection

**Grep Match Configuration:**
- **Pattern:** "New passwords do not match"
- **Purpose:** Identify when current password is correct

**Attack Type:** Sniper (single parameter enumeration)

**Payload Source:** Common password wordlist

**Attack Results:**

| Payload | Error Message | Grep Match | Analysis |
|---------|---------------|------------|----------|
| 123456 | Current password is incorrect | ✗ | Invalid |
| password | Current password is incorrect | ✗ | Invalid |
| admin | Current password is incorrect | ✗ | Invalid |
| **[password]** | **New passwords do not match** | **✓** | **Valid - Correct password!** |
| letmein | Current password is incorrect | ✗ | Invalid |

**Valid Password Identified:**

The unique response "New passwords do not match" with Grep Match hit confirms the correct current password was submitted.

**Attack Success:**
- ✓ Password enumeration successful
- ✓ Brute-force protection bypassed (no lockout triggered)
- ✓ Victim password discovered
- ✓ No authentication required for enumeration

### 8. Account Takeover Validation

Authenticated to victim account using discovered credentials to confirm complete compromise.

**Discovered Credentials:**
- Username: `carlos`
- Password: `[discovered_password]`
<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/6065b7b0-7778-4080-8ded-739eadc7eb5e" />

*Successful victim account access (right) | Authentication with discovered credentials captured in Intruder results (left)*

**Authentication Request:**
```http
POST /login HTTP/1.1

username=carlos&password=[discovered_password]
```

**Response:** 302 Found → Redirect to `/my-account`

**Complete Account Access:**
- ✓ Full authentication successful
- ✓ Carlos's username and email visible
- ✓ Email modification capability available
- ✓ Password change functionality accessible
- ✓ Complete control over victim account

**Lab completion confirmed: Password enumeration successful**

## Technical Impact

**Severity: High (CVSS 8.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerabilities:**

**1. CWE-204: Observable Response Discrepancy**
- Different error messages for valid vs invalid passwords
- Password validation state disclosed through error text
- Enables precise password enumeration

**2. CWE-640: Weak Password Recovery Mechanism**
- Password change logic fails to enforce consistent security controls
- Mismatched new passwords bypass logout protection
- Brute-force protection circumvented through logic exploitation

**3. CWE-307: Improper Restriction of Excessive Authentication Attempts**
- Protection only triggers on specific input patterns
- Alternate input patterns bypass lockout mechanism
- Insufficient validation of attack vectors

**Vulnerability Chain:**
```
Password Change Functionality
        ↓
Mismatched New Passwords Submitted
        ↓
Error Message Analysis
        ↓
Correct Password → "New passwords do not match"
Incorrect Password → "Current password is incorrect"
        ↓
Information Leakage Identified
        ↓
Bypass Logout Protection
        ↓
Automated Enumeration (No Lockout)
        ↓
Valid Password Discovered
        ↓
Complete Account Takeover
```

**Broken Security Logic:**

**Vulnerable Password Change Flow:**
```
1. Validate current password
   ├─ Valid → Check new passwords
   │         ├─ Match → Change password (success)
   │         └─ Mismatch → Error: "New passwords do not match" (LEAK)
   └─ Invalid → Check new passwords
             ├─ Match → Logout + Error: "Current password is incorrect"
             └─ Mismatch → Error: "Current password is incorrect" (NO LOGOUT)
```

**Security Flaws:**

1. **Information Disclosure:** Password validity revealed through error messages
2. **Inconsistent Enforcement:** Logout triggered only when new passwords match
3. **Bypass Opportunity:** Mismatched new passwords prevent logout
4. **Enumeration Enablement:** Brute-force protection ineffective

**Secure Implementation Requirements:**

**Generic Error Messages:**
- All password change failures → "Password change failed"
- No distinction between invalid current password vs mismatched new passwords
- Identical responses regardless of failure reason

**Consistent Session Management:**
- Invalid current password → Always terminate session
- Apply logout regardless of new password values
- No bypass paths through input manipulation

**Comprehensive Brute-Force Protection:**
- Rate limiting on password change attempts
- Account lockout after threshold (regardless of error type)
- CAPTCHA after multiple failures
- Monitoring for enumeration patterns

**Real-World Attack Impact:**

| Target | Consequence |
|--------|-------------|
| **Corporate Accounts** | Data exfiltration, privilege escalation |
| **Email Services** | Complete mailbox access, password reset for other services |
| **Banking** | Unauthorized transactions, account draining |
| **E-Commerce** | Fraudulent purchases, customer data theft |
| **Social Media** | Identity theft, reputation damage |

**Why This Attack is Effective:**

1. **No Authentication Required:** Attacker needs no valid credentials to start
2. **Silent Enumeration:** No alerts or lockouts during attack
3. **High Success Rate:** Common passwords found quickly
4. **Scalable:** Automated attacks against multiple accounts
5. **Low Detection:** Appears as normal password change attempts

**Compliance Violations:**
- OWASP Top 10 A07:2021 - Identification and Authentication Failures
- CWE/SANS Top 25 - Observable Discrepancy
- NIST SP 800-63B - Password Change Security Requirements
- PCI-DSS 8.2.1 - Account Enumeration Prevention

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic testing of password change logic
- Error message analysis for information disclosure
- Identification of protection bypass through input manipulation
- Automated enumeration using response pattern detection

**Critical Security Insights:**

**1. Error Messages Must Be Generic:**
All authentication-related failures must return identical errors:
- Never distinguish between invalid username, password, or other parameters
- Generic message: "Authentication failed" or "Invalid credentials"
- Consistent responses prevent information leakage

**2. State Transitions Must Be Consistent:**
Security controls must apply uniformly:
- Invalid password → Always logout (regardless of other inputs)
- No bypass paths through alternate input combinations
- Consistent enforcement across all failure scenarios

**3. Defense-in-Depth for Password Changes:**
Multiple layers prevent enumeration:
- Generic error messages (prevent information disclosure)
- Consistent session termination (prevent state manipulation)
- Rate limiting (prevent brute-force)
- Account lockout (limit attack attempts)
- CAPTCHA (detect automation)
- Monitoring (identify suspicious patterns)

**4. Logic Flaws Are Exploitable:**
Security vulnerabilities aren't always obvious bugs:
- Subtle behavioral differences enable attacks
- Input combination variations bypass protections
- Thorough testing required for all input permutations

**5. Password Change is Authentication:**
Password change operations are security-critical:
- Require current password verification
- Implement same protections as login
- Monitor for suspicious activity
- Apply rate limiting and lockout
- Consider requiring re-authentication for sensitive operations

**6. Testing Must Cover Edge Cases:**
Comprehensive security testing includes:
- Valid and invalid input combinations
- Mismatched parameters
- Boundary conditions
- State manipulation attempts
- All possible error paths

**Professional Skills Demonstrated:**
- Advanced logic flaw identification
- Error message pattern analysis
- Brute-force protection bypass techniques
- Automated enumeration attack configuration
- Understanding of authentication state management
- Systematic vulnerability assessment methodology

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-204 - Observable Response Discrepancy
4. CWE-640 - Weak Password Recovery Mechanism
5. CWE-307 - Improper Restriction of Excessive Authentication Attempts
6. OWASP Authentication Cheat Sheet
7. NIST SP 800-63B - Digital Identity Guidelines

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
