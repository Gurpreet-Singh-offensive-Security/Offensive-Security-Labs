# Lab 2: 2FA Simple Bypass via Broken Access Control

## Executive Summary

This lab demonstrates a critical authentication bypass vulnerability where two-factor authentication (2FA) can be circumvented through direct URL manipulation. Despite implementing 2FA as a security control, the application fails to enforce verification before granting access to authenticated resources. By exploiting this broken logic, I successfully bypassed the 2FA requirement and gained unauthorized access to victim accounts without possessing valid authentication codes.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | 2FA Bypass via Broken Access Control |
| **Risk**               | Critical - Complete Authentication Bypass |
| **Completion**         | December 28, 2025 |

## Objective

Bypass two-factor authentication controls through URL manipulation and access control flaws to achieve unauthorized access to victim accounts without valid 2FA codes.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception
- Email client interface for 2FA code retrieval

**Target:** Multi-stage authentication flow with 2FA enforcement

## Exploitation Walkthrough

### 1. Authentication Flow Analysis

Accessed the lab environment displaying "2FA simple bypass" challenge. Configured Burp Suite to intercept all authentication traffic for analysis.
<img width="1920" height="983" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/52a5c66b-c6eb-4fe7-b75d-440709c6bdb6" />

*Lab 2 environment - 2FA bypass vulnerability with Burp Suite Professional configured*

### 2. Legitimate Authentication Flow Testing

Authenticated with my credentials to understand the complete 2FA workflow and identify potential security gaps.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`
<img width="1920" height="983" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/afb8075b-31be-4503-8958-fa4ee0dd6598" />



*Login request intercepted - 302 redirect to 2FA verification page (left) | 4-digit security code prompt displayed (right)*

**Authentication Flow Observed:**
```
POST /login (credentials)
  ↓ 302 Found
GET /login2 (2FA verification page)
  ↓ User enters code
POST /login2 (verification)
  ↓ 302 Found
GET /my-account (authenticated session)
```

### 3. 2FA Code Retrieval

Accessed the email client to retrieve the 4-digit verification code sent to my account.
<img width="1920" height="983" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/f0a38273-31d4-4d10-b087-02c00ce914db" />

*Email client showing 2FA verification code: 1250*

**Code Received:** `1250`

### 4. 2FA Verification Completion

Submitted the verification code through the 2FA page and captured the complete authentication sequence.

<img width="1920" height="983" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/595952fb-f024-4c84-95fe-421c1db43286" />

*2FA verification request captured - mfa-code=1250 submitted*

### 5. Post-Authentication Access Confirmed

Successfully completed 2FA verification and gained access to authenticated account dashboard.

<img width="1920" height="983" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/e7e00be1-647b-4016-bbdb-3d3a5e2f193a" />


*Account dashboard accessed - email modification functionality visible*

**Legitimate Flow Confirmed:**
- ✅ Valid credentials required for initial login
- ✅ 2FA code sent to registered email
- ✅ Correct code verification required
- ✅ Access granted to `/my-account` after full verification

### 6. Victim Credential Testing

Logged out and attempted authentication with victim credentials to test 2FA enforcement against unauthorized access.

**Victim Credentials:**
- Username: `carlos`
- Password: `montoya`

<img width="1920" height="983" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/7cf9997a-afae-490c-b554-43a76b30681a" />

*Victim credentials validated - 302 redirect to 2FA page (left) | 2FA verification prompt reached without email access (right)*

**Critical Observation:** Successfully authenticated with victim's password and reached 2FA verification page. As an attacker, I do not have access to the victim's email to retrieve the 2FA code.

**Expected Security Behavior:** Application should block access to all authenticated resources until 2FA verification is completed.

### 7. 2FA Bypass via URL Manipulation

**Attack Hypothesis:** The application may fail to enforce 2FA verification before granting access to protected endpoints.

**Exploitation Technique:** 
Instead of submitting a 2FA code at `/login2`, directly navigated to the protected resource by manually changing the URL to `/my-account`.

<img width="1920" height="983" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/42586a97-3ff1-4fc7-a616-ca7e18e9e701" />

*URL manipulation executed - bypassing 2FA verification by directly accessing /my-account*

**Attack Flow Comparison:**

| Normal Flow | Attack Flow |
|-------------|-------------|
| /login → /login2 → /my-account | /login → ~~*/login2*~~ → /my-account |
| Password + 2FA Code Required | Password Only Required |

**Root Cause:** The application sets a valid session cookie after first-factor authentication (username/password) but fails to verify 2FA completion status before granting access to protected resources. This broken access control allows partially-authenticated users to access sensitive endpoints.

### 8. Successful Account Takeover

Direct navigation to `/my-account` succeeded without providing any 2FA code. Gained complete unauthorized access to victim account.

<img width="1920" height="983" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/1ca8103b-4862-447f-a01e-2896ce8384f3" />

*Victim account fully compromised - username, email, and account modification capabilities exposed*

**Compromise Achieved:**
- ✅ Full access to `carlos` account
- ✅ Email address visible and modifiable
- ✅ All account functionality accessible
- ✅ 2FA completely bypassed without valid code

**Security Impact:** Despite 2FA implementation, the control provides zero protection when enforcement is not applied to all authenticated resources. The vulnerability renders the entire 2FA system ineffective.

### 9. Lab Completion

<img width="1920" height="983" alt="LAB2_ss9 1" src="https://github.com/user-attachments/assets/fb2202c5-1d15-4e4e-bb84-25fe5dba97f8" />

*Lab solved successfully - "Congratulations, you solved the lab!"*

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-306: Missing Authentication for Critical Function**
- Application fails to verify 2FA completion before granting resource access
- Session established after first-factor authentication only
- No enforcement mechanism validates multi-factor authentication state

**Broken Authentication Logic:**

The application incorrectly assumes that reaching the 2FA page implies eventual verification, but fails to validate completion:
```
Vulnerable Logic:
- User enters password → Session created with 'authenticated' flag
- User redirected to /login2 → No enforcement on subsequent requests
- User can access /my-account → Only checks 'authenticated' flag, not '2FA verified' flag

Secure Logic:
- User enters password → Session marked as 'password_verified'
- User must complete 2FA → Session marked as 'fully_authenticated'
- Protected resources → Verify 'fully_authenticated' flag before access
```

**Real-World Attack Impact:**

| Sector | Impact |
|--------|--------|
| **Banking/Finance** | Unauthorized fund transfers, account draining |
| **Healthcare** | PHI exposure, HIPAA violations, medical record access |
| **E-Commerce** | Fraudulent purchases, payment method theft |
| **Corporate Systems** | Sensitive data access, privilege escalation |
| **Cloud Services** | Infrastructure compromise, data exfiltration |

**Compliance Violations:**
- PCI-DSS Requirement 8.3 (Multi-Factor Authentication) - Failed
- NIST SP 800-63B Authentication Guidelines - Non-compliant
- SOC 2 Trust Principles - Authentication controls ineffective

**Attack Complexity:** Trivial - requires only victim's password and basic URL manipulation. No specialized tools or advanced techniques needed.

## Remediation

### Immediate Fixes

**1. Enforce 2FA Verification on All Protected Endpoints**

Every authenticated resource must verify complete 2FA status:
```python
# Check both password AND 2FA verification
if not session.get('mfa_verified'):
    return redirect('/login2')
```

**2. Implement Session State Tracking**

Track authentication progress through distinct session states:
- `UNAUTHENTICATED` - No credentials provided
- `PASSWORD_VERIFIED` - First factor complete
- `FULLY_AUTHENTICATED` - Both factors verified

**3. Apply Centralized Access Control**

Use middleware or decorators to consistently enforce 2FA across all endpoints:
```python
@require_full_authentication
def my_account():
    return render_account_page()
```

**4. Time-Limit 2FA Windows**

Require 2FA completion within 5 minutes of password verification to prevent session lingering.

### Long-Term Solutions

**1. Framework-Level Protection:**
Utilize security frameworks with built-in 2FA enforcement (e.g., django-two-factor-auth, passport-2fa) that handle state management automatically.

**2. Security Monitoring:**
- Log all access attempts to protected resources
- Alert on authenticated sessions accessing resources without 2FA completion
- Track and investigate 2FA bypass attempts

**3. Security Testing:**
- Include broken access control testing in security assessments
- Automated tests for authentication state enforcement
- Regular penetration testing of multi-factor authentication flows

**4. Defense-in-Depth:**
- Rate limit 2FA attempts
- Implement account lockout after multiple failed 2FA submissions
- Add security headers to prevent caching of authenticated pages
- Monitor for suspicious authentication patterns

## Key Takeaways

**Penetration Testing Skills:**
- Systematic analysis of multi-stage authentication flows
- Identification of access control enforcement gaps
- URL manipulation techniques for bypass testing
- Clear documentation of broken authentication logic

**Critical Security Insights:**

**1. Implementation ≠ Enforcement**
Simply implementing 2FA is worthless without proper enforcement. Every protected endpoint must validate complete authentication state.

**2. Session State Management is Critical**
Applications must track authentication progress and verify completion before granting access. Partial authentication should never provide resource access.

**3. Broken Access Control Extends to Auth States**
Access control vulnerabilities aren't limited to data objects—they apply equally to authentication states and must be validated consistently.

**4. Defense-in-Depth Requires Consistency**
Security controls must be applied uniformly across all endpoints. A single unprotected route undermines the entire security model.

**Professional Application:**
- Demonstrates understanding of authentication flow vulnerabilities
- Shows ability to identify logic flaws in security implementations
- Provides clear impact assessment and remediation guidance
- Highlights importance of consistent security control enforcement

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. OWASP ASVS - V2: Authentication Requirements
4. CWE-306 - Missing Authentication for Critical Function
5. NIST SP 800-63B - Digital Identity Guidelines
6. PCI-DSS Requirement 8.3 - Multi-Factor Authentication

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
