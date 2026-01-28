# Lab 13: Broken Brute-Force Protection via Multiple Credentials

## Executive Summary

This lab demonstrates a critical vulnerability in JSON-based authentication where improper input validation enables array-based credential submission. By submitting hundreds of password candidates in a single JSON array, I completely bypassed brute-force protection mechanisms and achieved instant account compromise without triggering rate limiting or lockout controls. This vulnerability showcases the severe risks of inadequate API input validation and the importance of proper authentication attempt tracking.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | JSON Array Injection in Authentication |
| **Risk**               | Critical - Instant Authentication Bypass |
| **Completion**         | January 3, 2026 |

## Objective

Exploit JSON-based authentication to submit multiple password candidates in a single request, bypassing brute-force protection through array parameter manipulation and achieving unauthorized account access.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for JSON manipulation
- Browser Developer Tools for session hijacking

**Target:** JSON authentication endpoint with inadequate input validation

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed lab environment displaying "Broken brute-force protection, multiple credentials per request" challenge and configured Burp Suite Professional for API testing.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/c8a7d2c0-8176-47d5-b868-c64198d44cc9" />

*Lab 13 environment with Burp Suite Professional configured*

### 2. Authentication Flow Analysis

Authenticated with test credentials to analyze request structure and identify the authentication mechanism.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/746b6c68-e834-4b1f-a43d-cdcb87196a19" />

*Account accessed showing email update functionality (right) | JSON authentication captured in HTTP history (left)*

**Authentication Request Captured:**
```http
POST /login HTTP/1.1
Content-Type: application/json

{
  "username": "wiener",
  "password": "peter"
}
```

**Critical Observation:** Authentication uses JSON format with structured data submission. JSON parsers often accept flexible data types, potentially allowing arrays or objects instead of simple strings.

### 3. Brute-Force Protection Testing

Sent authentication request to Burp Repeater and tested security controls with incorrect credentials.

<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/7cd50662-61aa-4030-9223-738a7ba74576" />

*Account locked after 2 failed login attempts*

**Test Results:**
- First incorrect attempt: Accepted
- Second incorrect attempt: Account locked

**Protection Mechanism:** 2-attempt threshold triggers temporary account lockout, preventing traditional brute-force attacks.

### 4. JSON Array Injection Discovery

**Attack Hypothesis:** JSON parsers may accept arrays for parameters expecting strings. Tested if password parameter accepts array values.

**Modified Request:**
```http
POST /login HTTP/1.1
Content-Type: application/json

{
  "username": "wiener",
  "password": ["peter", "123456"]
}
```
<img width="1920" height="983" alt="LAB1_ss3 1" src="https://github.com/user-attachments/assets/f3336514-4970-4084-bd61-ca1dc9580f19" />

*Array injection test - 302 response confirms successful authentication*

**Response:** 302 Found - Redirect to `/my-account`

**Critical Vulnerability Confirmed:**

The application:
1. Accepts password parameter as JSON array
2. Iterates through array values
3. Authenticates successfully if any password matches
4. Counts as single authentication request
5. Bypasses brute-force protection completely

**Attack Vector:** Submit entire password wordlist in single array, achieving instant brute-force without triggering lockout.

### 5. Mass Password Enumeration Attack

Constructed comprehensive attack payload containing complete password wordlist in JSON array format.

<img width="1920" height="983" alt="LAB1_ss4 1" src="https://github.com/user-attachments/assets/4b12e174-cbb5-46fd-8814-b32484b7792d" />

*Password array payload - beginning of wordlist*

<img width="1920" height="983" alt="LAB1_ss4 2" src="https://github.com/user-attachments/assets/4c18198e-f158-45c2-aa5d-e75536551234" />

*Password array payload - complete wordlist included*

**Attack Payload:**
```json
{
  "username": "carlos",
  "password": [
    "123456",
    "password",
    "12345678",
    "qwerty",
    "abc123",
    "monkey",
    "letmein",
    "trustno1",
    ... [hundreds of password candidates]
  ]
}
```

**Payload Characteristics:**
- **Target:** `carlos` (victim username)
- **Passwords:** Complete common password wordlist
- **Array Size:** Hundreds to thousands of candidates
- **Request Count:** Single request
- **Protection Bypass:** Complete

### 6. Authentication Success & Session Capture

Executed attack payload and captured successful authentication response.
<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/71fcdbe5-5ad5-4e4e-ae81-0e0f44faed89" />

*Attack successful - 302 response with valid session token*

**Response Received:**
```http
HTTP/1.1 302 Found
Location: /my-account
Set-Cookie: session=[CARLOS_SESSION_TOKEN]; HttpOnly
```

**Attack Success:**
- ✅ One password in array matched victim's password
- ✅ Valid session token issued for Carlos's account
- ✅ Single request executed (no lockout triggered)
- ✅ Brute-force protection completely bypassed

**Session Token Extracted:** Captured from Set-Cookie header for session hijacking.

### 7. Session Hijacking Preparation

Accessed my own account page and prepared for session token replacement.
<img width="1920" height="983" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/d3a420b4-f44d-443c-be8e-a078a9abbd38" />

*Browser DevTools showing session cookie (right) | Attack response with Carlos's token (left)*

**Session Hijacking Steps:**
1. Opened Browser Developer Tools → Application → Cookies
2. Located session cookie for application
3. Replaced value with Carlos's captured session token

**Cookie Modification:**
```
Original: session=[MY_SESSION_TOKEN]
Modified: session=[CARLOS_SESSION_TOKEN]
```

### 8. Account Takeover Confirmation

Modified URL to access victim account using hijacked session token.

**URL Navigation:**
- Changed from: `/login`
- Changed to: `rest url with deleted /login`

<img width="1920" height="983" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/3a87519d-2ae5-40cf-93d9-8a83194671d7" />

*Victim account fully accessed (right) | Session request captured in Repeater (left)*

**Complete Account Access:**
- ✅ Full authentication bypass successful
- ✅ Carlos's username and email visible
- ✅ Email modification capability available
- ✅ Complete control over victim account
- ✅ Single-request attack with minimal forensic footprint

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**1. CWE-1286: Improper Validation of Array Index**
- JSON parser accepts arrays where strings expected
- No validation of parameter types or array sizes
- Enables batch password testing in single request

**2. CWE-307: Improper Restriction of Excessive Authentication Attempts**
- Protection counts requests, not password validation attempts
- Array-based submission bypasses attempt counters
- Lockout mechanism rendered ineffective

**3. CWE-20: Improper Input Validation**
- Missing schema validation on JSON input
- No strict type checking enforced
- Parameter type flexibility exploitable

**Attack Chain:**
```
JSON Authentication Endpoint
        ↓
Password Parameter Type Testing
        ↓
Array Injection Accepted
        ↓
Multiple Passwords in Single Request
        ↓
Application Iterates Array
        ↓
First Valid Password → Success
        ↓
Session Token Issued
        ↓
Single Request (No Protection Triggered)
        ↓
Complete Account Takeover
```

**Attack Comparison:**

| Method | Requests | Passwords Tested | Detection Risk | Time Required |
|--------|----------|------------------|----------------|---------------|
| Traditional Brute-Force | 10,000 | 10,000 | High (10,000 failed logins) | Hours/Days |
| **Array Injection** | **1** | **10,000** | **Minimal (single request)** | **Seconds** |

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **Corporate Systems** | Employee account takeover, data exfiltration |
| **Financial Services** | Unauthorized transactions, account draining |
| **E-Commerce** | Fraudulent purchases, customer data theft |
| **Healthcare** | PHI exposure, medical record manipulation |
| **SaaS Platforms** | Business data access, service disruption |

**Why This Attack is Severe:**

1. **Instant Execution:** Entire password list tested in seconds
2. **Detection Evasion:** Single request appears as normal authentication
3. **No Lockout:** Brute-force protection completely bypassed
4. **Scalability:** Easily automated against multiple accounts
5. **Minimal Forensics:** Single log entry vs. thousands of failures

## Key Takeaways

**Penetration Testing Methodology:**
- API parameter type testing (strings vs. arrays vs. objects)
- JSON schema validation assessment
- Brute-force protection bypass techniques
- Session token extraction and replay attacks

**Critical Security Insights:**

**1. Strict Type Validation is Mandatory:**
API endpoints must enforce parameter types:
- Define expected types in JSON schema
- Reject arrays when strings expected
- Validate all input against formal schema
- Implement server-side type checking

**2. Count Attempts, Not Requests:**
Effective brute-force protection requires:
- Password validation attempt counting
- Array size limits for all parameters
- Per-username attempt tracking
- Time-based lockouts independent of request count

**3. Defense-in-Depth for Authentication:**
Multiple security layers prevent bypass:
- Input validation (reject unexpected types)
- Rate limiting (count password validations)
- Account lockout (track attempts per username)
- CAPTCHA (after threshold)
- Multi-factor authentication

**4. API Security Requires Special Attention:**
JSON/REST APIs face unique challenges:
- Flexible data structures enable creative attacks
- Traditional WAF rules may not detect array-based attacks
- Schema validation often overlooked in development
- Type coercion vulnerabilities common in JSON parsers

**5. Session Management Remains Critical:**
Even with authentication bypass:
- Implement short session timeouts
- Use secure cookie flags (HttpOnly, Secure, SameSite)
- Consider session binding to IP/device
- Monitor for unusual session activity

**Professional Skills Demonstrated:**
- Advanced API vulnerability exploitation
- JSON parameter manipulation techniques
- Brute-force protection bypass methodology
- Session hijacking and token replay
- Understanding of authentication security architecture
- Comprehensive attack documentation

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. OWASP API Security Top 10 - API2:2023 Broken Authentication
4. CWE-1286 - Improper Validation of Array Index
5. CWE-307 - Improper Restriction of Excessive Authentication Attempts
6. NIST SP 800-63B - Digital Identity Guidelines

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
