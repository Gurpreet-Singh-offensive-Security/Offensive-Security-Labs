# Lab 3: Password Reset Broken Logic

## Executive Summary

This lab demonstrates a critical authentication vulnerability in the password reset functionality where token validation is improperly implemented. By analyzing the password reset flow, I discovered that the application fails to validate the temporary reset token, allowing arbitrary password changes for any user account. This broken logic enabled complete account takeover of the victim without requiring access to their email or any valid reset token.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | Password Reset Broken Logic |
| **Risk**               | Critical - Arbitrary Account Takeover |
| **Completion**         | December 28, 2025 |

## Objective

Exploit broken password reset logic to change victim's password without valid reset token, achieving unauthorized account access through authentication bypass.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception
- Burp Repeater for request manipulation
- Email client for reset token retrieval

**Target:** Password reset functionality at `/forgot-password` endpoint

## Exploitation Walkthrough

### 1. Initial Setup & Reconnaissance

Accessed the lab environment displaying "Password reset broken logic" challenge. Configured Burp Suite to intercept password reset flow.
<img width="1920" height="983" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/5d6b2f6e-a664-47b6-bccf-c381e7b5517e" />

*Lab 3 environment - Password reset vulnerability with Burp Suite Professional configured*

### 2. Password Reset Flow Initiation

Attempted login with incorrect credentials to trigger the "Forgot password?" functionality and analyze the reset process.

<img width="1920" height="983" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/ed0d6dd0-75bc-47e4-8927-5d55b8d078eb" />

*Forgot password request initiated (left - Burp Proxy) | Password reset link clicked (right - Lab interface)*

**Request Captured:**
```http
GET /forgot-password HTTP/1.1
Host: [lab-id].web-security-academy.net
```

### 3. Username Submission for Password Reset

Submitted my username (`wiener`) to test the legitimate password reset flow and understand token generation.

<img width="1920" height="983" alt="LAB3_ss3" src="https://github.com/user-attachments/assets/b0ac73cf-c876-4a0f-b4fa-fd6e5c12e184" />

*Password reset username submission intercepted - POST request with username=wiener*

**Request Details:**
```http
POST /forgot-password HTTP/1.1

username=wiener
```

**Flow Observed:** Application accepts username and initiates password reset process without additional verification.

### 4. Reset Token Delivery

Accessed email client to retrieve the password reset link containing the temporary token.

<img width="1920" height="983" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/40d68f46-0d0f-40e0-abf8-4bd2598b031c" />

*Email received with password reset link containing temporary token*

**Reset Link Format:**
```
https://[lab-id].web-security-academy.net/forgot-password?temp-forgot-password-token=[TOKEN]
```

### 5. Password Reset Request Analysis

Clicked the reset link and submitted new password to capture the complete reset request structure.
<img width="1920" height="983" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/65834d0f-d478-4e10-a4dc-d980a2b6cb43" />

*Password reset submission captured - includes username, new passwords, and temp-forgot-password-token*

**Complete Reset Request:**
```http
POST /forgot-password HTTP/1.1

temp-forgot-password-token=[VALID_TOKEN]&username=wiener&new-password-1=newpass&new-password-2=newpass
```

**Parameters Identified:**
- `temp-forgot-password-token` - Temporary reset token from email
- `username` - Target account username
- `new-password-1` - New password
- `new-password-2` - Password confirmation

### 6. Token Validation Testing

Sent the password reset request to Burp Repeater for manipulation and validation testing.

<img width="1920" height="983" alt="LAB3_ss6" src="https://github.com/user-attachments/assets/82921512-1fb9-4070-9951-a3e4247c58b4" />

*Request sent to Repeater - 302 Found response confirms successful password change with valid token*

**Test Result:** Legitimate reset with valid token returns `302 Found`, confirming password change success.

### 7. Token Requirement Testing

**Critical Test:** Removed the `temp-forgot-password-token` parameter entirely to determine if token validation is enforced.

<img width="1920" height="983" alt="LAB3_ss7" src="https://github.com/user-attachments/assets/f39875c5-4bcf-497d-9787-49f36f267960" />

*Token parameter removed - 302 Found response received, indicating no token validation*

**Modified Request:**
```http
POST /forgot-password HTTP/1.1

username=wiener&new-password-1=newpass&new-password-2=newpass
```

**Critical Finding:** Application returned `302 Found` without any token present. This confirms the application does not validate the reset token before processing password changes.

**Vulnerability Confirmed:** The password reset functionality accepts username and new password parameters without verifying possession of a valid reset token. This broken logic allows arbitrary password changes for any user account.

### 8. Victim Account Takeover

Exploited the broken logic to change the victim's password without any valid reset token.

**Attack Execution:**

Modified the password reset request in Burp Repeater:
- Changed `username` to victim account: `carlos`
- Set new password: `peter1`
- Omitted `temp-forgot-password-token` parameter entirely

<img width="1920" height="983" alt="LAB3_ss8 1" src="https://github.com/user-attachments/assets/c22b37dd-7d6a-4520-bdac-651b39787ae2" />

*Victim password changed - username=carlos with new-password=peter1, no token required (302 Found response)*

**Malicious Request:**
```http
POST /forgot-password HTTP/1.1

username=carlos&new-password-1=peter1&new-password-2=peter1
```

**Response:** `302 Found` - Password successfully changed for victim account.

### 9. Account Access Confirmed

Authenticated to victim account using the newly set password to confirm complete account compromise.

<img width="1920" height="983" alt="LAB3_ss8 2" src="https://github.com/user-attachments/assets/bcd24361-fc0d-407c-9687-1ad244754a6d" />

*Victim account accessed successfully - full control with email modification capability | Lab solved*

**Credentials Used:**
- Username: `carlos`
- Password: `peter1` (attacker-controlled)

**Compromise Achieved:**
- ✅ Victim password changed without email access
- ✅ No reset token required
- ✅ Full account access obtained
- ✅ Email modification capability available
- ✅ Complete account takeover successful

**Lab completion message displayed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-640: Weak Password Recovery Mechanism for Forgotten Password**
- Password reset functionality fails to validate reset tokens
- Username parameter accepted without verification
- No correlation between token and user account
- Arbitrary password changes possible without authorization

**Broken Password Reset Logic:**
```
Vulnerable Implementation:
1. User requests password reset → Token sent to email
2. User submits new password with token → Application ignores token
3. Application changes password based solely on username parameter
4. No validation that requester possesses valid token

Secure Implementation:
1. User requests password reset → Token generated and sent to email
2. User submits new password with token → Application validates token
3. Application verifies token matches user and hasn't expired
4. Password changed only if valid token provided
```

**Attack Methodology:**

| Step | Action | Result |
|------|--------|--------|
| 1 | Intercept legitimate password reset request | Identify request structure |
| 2 | Remove token parameter | 302 response - no validation |
| 3 | Change username to victim | Password changed for victim |
| 4 | Login with new credentials | Complete account takeover |

**Security Implications:**

**1. Mass Account Compromise:**
- Attacker can change passwords for any known username
- No email access required
- No rate limiting observed
- Automated attacks possible against entire user base

**2. Privilege Escalation:**
- Administrative accounts vulnerable
- High-value targets easily compromised
- No additional verification for privileged users

**3. Persistent Access:**
- Attacker-controlled passwords enable long-term access
- Legitimate user locked out of their own account
- Password changes undetectable without monitoring

**Real-World Attack Scenarios:**

| Sector | Impact |
|--------|--------|
| **Financial Services** | Account draining, unauthorized transactions |
| **Healthcare** | Patient data manipulation, HIPAA violations |
| **E-Commerce** | Fraudulent purchases, customer data theft |
| **SaaS Platforms** | Data exfiltration, service disruption |
| **Corporate Systems** | Intellectual property theft, espionage |

**Compliance Violations:**
- OWASP ASVS V2.5 - Password Reset Requirements
- PCI-DSS Requirement 8.2.4 - Password Reset Procedures
- NIST SP 800-63B Section 5.1.1.2 - Authentication Recovery

**Attack Complexity:** Trivial - requires only knowledge of victim's username and ability to send HTTP requests. No specialized tools or sophisticated techniques needed.

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic analysis of password reset workflows
- Token validation testing through parameter manipulation
- Identification of missing authentication checks
- Demonstration of full attack chain from discovery to compromise

**Critical Security Principles:**

**1. Token Validation is Mandatory:**
Password reset tokens must be cryptographically validated before processing any password change. The token serves as proof that the requester controls the associated email account.

**2. Parameter Trust is Dangerous:**
Applications must never trust client-supplied parameters (like username) without proper authorization. The username should be derived from the validated token, not from user input.

**3. Defense-in-Depth for Sensitive Operations:**
Password changes are high-risk operations requiring multiple layers of validation:
- Valid reset token verification
- Token-to-user correlation
- Token expiration checks
- Rate limiting
- Notification to account holder

**4. Security by Design:**
Password reset functionality must be designed with security as the primary concern. Convenience cannot compromise account security.

**Professional Skills Demonstrated:**
- Methodical security assessment of authentication flows
- Request manipulation and parameter testing
- Clear documentation of vulnerability exploitation
- Impact assessment with practical attack demonstration
- Understanding of authentication security principles

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. OWASP ASVS - V2.5: Password Recovery Requirements
4. CWE-640 - Weak Password Recovery Mechanism
5. NIST SP 800-63B - Digital Identity Guidelines
6. OWASP Forgot Password Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
