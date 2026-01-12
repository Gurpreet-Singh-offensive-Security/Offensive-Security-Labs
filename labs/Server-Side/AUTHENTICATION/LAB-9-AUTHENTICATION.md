# Lab 9: Brute-Forcing a Stay-Logged-In Cookie

## Executive Summary

This lab demonstrates a critical vulnerability in "stay logged in" functionality where session persistence relies on predictable, client-side cookie values. The application generates stay-logged-in cookies using a reversible encoding scheme (Base64) containing usernames and MD5-hashed passwords. By reverse-engineering the cookie generation algorithm and automating password hash enumeration, I successfully brute-forced victim credentials through cookie manipulation, achieving direct account access without traditional authentication. This showcases the dangers of weak cryptographic practices and predictable session management.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Weak Cryptographic Implementation in Stay-Logged-In Cookies |
| **Risk**               | High - Direct Account Access via Cookie Manipulation |
| **Completion**         | January 3, 2026 |

## Objective

Reverse-engineer stay-logged-in cookie generation mechanism, automate password brute-forcing through cookie reconstruction, and achieve unauthorized account access by exploiting weak cryptographic practices.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Intruder with payload processing
- Burp Inspector for cookie analysis
- Base64 and MD5 hash manipulation

**Target:** Stay-logged-in cookie functionality with weak cryptographic implementation

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed the lab environment displaying "Brute-forcing a stay-logged-in cookie" challenge. Configured Burp Suite Professional for cookie analysis and manipulation.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/4fba436e-6059-43f7-b537-759a53e320e0" />

*Lab 9 environment - Stay-logged-in cookie vulnerability title in lab (right) with Burp Suite Professional configured(left)*

### 2. Legitimate Authentication & Cookie Capture

Authenticated with my credentials while enabling "stay logged in" option to capture and analyze cookie generation.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`
- Option: Stay logged in ✓
<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/862b4b2c-720c-4705-9682-e401320ad6c1" />

*Account accessed with stay-logged-in enabled (right) | HTTP requests captured showing stay-logged-in cookie (left)*

**Request Captured:**
```http
GET /my-account?id=wiener HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: session=[SESSION_TOKEN]; stay-logged-in=[ENCODED_VALUE]
```

**Cookie Identified:**
- Cookie Name: `stay-logged-in`
- Cookie Value: Encoded string (appears to be Base64)
- Purpose: Persistent authentication without re-login

**Observation:** The stay-logged-in cookie persists authentication across sessions, suggesting it contains sufficient information to authenticate users without password verification.

### 3. Cookie Structure Reverse Engineering

Used Burp Inspector to decode and analyze the stay-logged-in cookie structure.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/7f73e298-f9dd-4f44-a2dd-085608e427ae" />

*Stay-logged-in cookie selected and analyzed in Burp Inspector - Base64 decoded revealing username:MD5_hash format*

**Cookie Analysis:**

**Encoded Value (Base64):**
```
d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw
```

**Decoded Value:**
```
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

**Structure Identified:**
```
Format: username:md5_hash
- Username: wiener
- Separator: :
- Hash: 51dc30ddc473d43a6011e9ebba6ca770 (32 characters = MD5)
```

**Hash Verification:**

MD5 hash of password "peter":
```bash
echo -n "peter" | md5sum
51dc30ddc473d43a6011e9ebba6ca770
```

**Cookie Generation Algorithm Discovered:**
```
cookie_value = base64_encode(username + ":" + md5(password))
```

**Vulnerability Identified:**

1. **Weak Hashing:** MD5 is cryptographically broken and fast to compute
2. **Predictable Format:** Simple concatenation of username and hash
3. **No Salt:** Password hashing without salt enables rainbow table attacks
4. **No HMAC:** No server-side signature to prevent tampering
5. **Client-Side Trust:** Application trusts cookie for authentication

**Attack Vector:** Generate valid stay-logged-in cookies by brute-forcing MD5 password hashes.

### 4. Cookie Brute-Force Proof of Concept

Validated the attack methodology using my own credentials before targeting the victim account.

**Attack Configuration in Burp Intruder:**

Sent GET /my-account request to Intruder and configured payload processing to generate valid cookies.
<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/8a417fb3-c735-46d6-806f-a5d6d8c71497" />

*Proof of concept attack configured - Password payload with hash processing (right) | Successful authentication via crafted cookie (middle)*

**Intruder Setup:**

**Target Position:**
```http
GET /my-account?id=wiener HTTP/1.1
Cookie: stay-logged-in=§payload§
```

**Payload Processing Rules (in order):**

1. **Hash: MD5**
   - Converts password to MD5 hash
   - Example: `peter` → `51dc30ddc473d43a6011e9ebba6ca770`

2. **Add prefix: wiener:**
   - Prepends username and separator
   - Example: `51dc30ddc473d43a6011e9ebba6ca770` → `wiener:51dc30ddc473d43a6011e9ebba6ca770`

3. **Encode: Base64**
   - Encodes final string to Base64
   - Example: `wiener:51dc30ddc473d43a6011e9ebba6ca770` → `d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw`

**Grep Match Configuration:**
- Pattern: "Update email"
- Purpose: Detect successful account access in responses

**Test Payload:**
- Single password: `peter` (my known password)

**Test Result:**

| Payload | Processed Cookie | Status | Response Length | Grep Match |
|---------|------------------|--------|-----------------|------------|
| peter | d2llbmVyOjUx...Nzcw | 200 | 3346 | ✓ Update email |

**Successful Response:**
- Account dashboard displayed in response
- "Update email" functionality present
- Direct account access achieved via cookie alone

**Attack Methodology Validated:** The application accepts crafted stay-logged-in cookies without additional verification, enabling password brute-forcing through cookie generation.

### 5. Victim Account Targeting

Reconfigured attack to target victim account using the validated methodology.

**Modified Attack Configuration:**

**Target Request:**
```http
GET /my-account?id=carlos HTTP/1.1
Cookie: stay-logged-in=§payload§
```

**Payload Processing Rules Updated:**

1. Hash: MD5
2. **Add prefix: carlos:** ← Changed to victim username
3. Encode: Base64

**Payload Source:**
- Common password wordlist
- Multiple passwords for brute-force enumeration

<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/e921e497-457f-4d47-997b-63a06cccc209" />

*Victim account brute-force attack - Password list processed into valid cookies | Successful password identified by response length difference*

**Attack Execution:**

Intruder generated stay-logged-in cookies for each password:
```
Password: 123456
  → MD5: e10adc3949ba59abbe56e057f20f883e
  → Prefix: carlos:e10adc3949ba59abbe56e057f20f883e
  → Base64: Y2FybG9zOmUxMGFkYzM5NDliYTU5YWJiZTU2ZTA1N2YyMGY4ODNl

Password: password
  → MD5: 5f4dcc3b5aa765d61d8327deb882cf99
  → Prefix: carlos:5f4dcc3b5aa765d61d8327deb882cf99
  → Base64: Y2FybG9zOjVmNGRjYzNiNWFhNzY1ZDYxZDgzMjdkZWI4ODJjZjk5

... (continued for entire wordlist)
```

**Attack Results:**

| Payload | Response Length | Grep Match | Analysis |
|---------|-----------------|------------|----------|
| 123456 | 173 | ✗ | Failed - Invalid credentials page |
| password | 173 | ✗ | Failed - Invalid credentials page |
| admin | 173 | ✗ | Failed - Invalid credentials page |
| **[password]** | **3346** | **✓ Update email** | **Success - Account dashboard** |
| letmein | 173 | ✗ | Failed - Invalid credentials page |

**Successful Credential Identified:**

- **Response Length:** 3346 bytes (significantly different from 173 bytes)
- **Grep Match:** "Update email" present
- **Response Content:** Full account dashboard for Carlos

**Valid Password Discovered:** By analyzing the unique response length and content match, the correct password was identified.

### 6. Account Takeover Confirmation

Validated complete account access by examining the successful Intruder response.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/e17da8c4-2484-4e5f-8f7a-7f91763d0877" />

*Intruder attack results showing successful payload (left) | Carlos account fully accessed via crafted cookie - Lab solved (right)*

**Successful Attack Response:**
```http
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 3346

[Account Dashboard HTML]
- Username: carlos
- Email: [carlos_email]
- Update email functionality
- Complete account access
```

**Complete Account Compromise:**
- ✅ Stay-logged-in cookie structure reverse-engineered
- ✅ Cookie generation algorithm replicated
- ✅ Password brute-forced via MD5 hash enumeration
- ✅ Valid cookie crafted for victim account
- ✅ Direct authentication bypass achieved
- ✅ Full account access without password entry
- ✅ Email modification capability available

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: High (CVSS 8.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerabilities:**

**1. CWE-327: Use of a Broken or Risky Cryptographic Algorithm**
- MD5 hashing for password storage
- Fast computation enables rapid brute-forcing
- No computational cost for attackers

**2. CWE-916: Use of Password Hash With Insufficient Computational Effort**
- Single MD5 iteration without salt
- No key derivation function (bcrypt, scrypt, Argon2)
- Rainbow table attacks feasible

**3. CWE-565: Reliance on Cookies without Validation**
- Application trusts client-provided stay-logged-in cookie
- No server-side validation or signature
- Cookie tampering enables arbitrary account access

**4. CWE-330: Use of Insufficiently Random Values**
- Predictable cookie generation algorithm
- No cryptographic randomness in session tokens
- Deterministic relationship between password and cookie

**Vulnerability Chain:**
```
Weak Cryptographic Implementation
        ↓
Capture Stay-Logged-In Cookie
        ↓
Reverse Engineer Cookie Format
        ↓
Identify: Base64(username:MD5(password))
        ↓
Automate Cookie Generation
        ↓
Brute-Force Password Hashes
        ↓
Valid Cookie Crafted
        ↓
Direct Account Access
```

**Why MD5 is Catastrophically Weak:**

| Hash Algorithm | Hash Rate (GPU) | Time to Crack 8-char password |
|----------------|-----------------|-------------------------------|
| MD5 | ~200 billion/sec | **Minutes** |
| SHA-1 | ~150 billion/sec | Minutes |
| bcrypt (cost 12) | ~50,000/sec | **Years** |
| Argon2id | ~10,000/sec | **Decades** |

**Attack Statistics:**

- **MD5 Hash Rate:** Modern GPUs compute ~200 billion MD5 hashes/second
- **Common Password Success:** Top 10,000 passwords = 99% of attacks succeed
- **Rainbow Tables:** Pre-computed MD5 hashes for billions of passwords freely available
- **Time to Crack:** Entire rockyou.txt wordlist (14 million passwords) in under 1 second

**Real-World Impact:**

| Scenario | Consequence |
|----------|-------------|
| **Password Database Breach** | All passwords instantly cracked |
| **Cookie Interception** | Permanent account access |
| **Mass Account Compromise** | Automated attacks against entire user base |
| **Credential Stuffing** | Cracked passwords used on other services |

**Compliance Violations:**
- OWASP ASVS V2.4 - Password Storage Requirements
- PCI-DSS 8.2.3 - Strong Cryptography Required
- NIST SP 800-63B - Password Hashing Standards
- GDPR Article 32 - Security of Processing

## Key Takeaways

**Penetration Testing Methodology:**
- Cookie structure reverse engineering
- Cryptographic weakness identification
- Automated hash generation and brute-forcing
- Payload processing for complex attack chains
- Pattern recognition in authentication mechanisms

**Critical Security Insights:**

**1. MD5 is Completely Broken for Security:**
MD5 must never be used for password hashing:
- Computationally trivial to brute-force
- Rainbow tables eliminate need for computation
- No security benefit over plaintext storage
- Use bcrypt, scrypt, or Argon2 instead

**2. Stay-Logged-In Requires Strong Design:**
Persistent authentication needs secure implementation:
- Never encode credentials in cookies
- Use cryptographically random session tokens
- Store token-to-user mapping server-side
- Implement token expiration and rotation
- Sign cookies with HMAC to prevent tampering

**3. Client-Side Data Cannot Be Trusted:**
All client-provided data must be validated:
- Cookies can be manipulated arbitrarily
- Encoding (Base64) is not encryption
- Server-side validation mandatory
- Cryptographic signatures required for integrity

**4. Defense-in-Depth for Session Management:**
Multiple layers prevent cookie-based attacks:
- Strong password hashing (bcrypt, Argon2)
- Cryptographically random tokens
- Server-side session storage
- HMAC signatures
- Secure cookie flags (HttpOnly, Secure, SameSite)
- Token expiration and rotation

**5. Cryptographic Choices Have Massive Impact:**
Algorithm selection determines security:
- MD5: Broken, never use
- SHA-1: Broken, never use
- SHA-256: Fast, inappropriate for passwords
- bcrypt/scrypt/Argon2: Correct choice for passwords

**Professional Skills Demonstrated:**
- Cryptographic vulnerability analysis
- Cookie manipulation and reverse engineering
- Automated attack tool configuration (payload processing)
- Understanding of hashing algorithms and weaknesses
- Clear documentation of cryptographic flaws

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-327 - Use of a Broken Cryptographic Algorithm
4. CWE-916 - Use of Password Hash With Insufficient Computational Effort
5. NIST SP 800-63B - Password Storage Requirements
6. OWASP Password Storage Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
