# Lab 1: Username Enumeration via Response Discrepancies

## Executive Summary

This lab demonstrates a critical authentication vulnerability where inconsistent server responses enable username enumeration and credential brute-forcing. By analyzing response length variations and error message differences, I successfully identified valid usernames, conducted a targeted password attack, and achieved complete account compromise. This exercise showcases systematic reconnaissance, automated exploitation, and the real-world impact of authentication implementation flaws.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | Username Enumeration via Response Discrepancies |
| **Risk**               | High - Authentication Bypass & Account Takeover |
| **Completion**         | December 28, 2025 |

## Objective

Exploit verbose error messaging to enumerate valid usernames, then leverage automated brute-force techniques to identify correct passwords and gain unauthorized access to user accounts.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Intruder for automated attacks
- Burp Repeater for validation

**Target:** `/login` endpoint with POST-based authentication

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment and configured Burp Suite Professional to intercept authentication traffic. Established baseline understanding of the login mechanism through manual testing.
<img width="1920" height="983" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/cf7450d7-b94e-45a2-a848-28e89297b08d" />

*Lab environment configured with Burp Suite Professional ready for traffic analysis*

### 2. Authentication Flow Analysis

Submitted test credentials (`don:123456`) to capture the authentication request and analyze server behavior. Identified critical security weakness: credentials transmitted in plaintext POST body without client-side encryption.
<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/f0f0bd67-e853-4d17-ae54-13afbbbc8e03" />

*POST request captured - credentials visible in plaintext (username=don&password=123456)*

**Key Finding:** Unlike modern applications that implement client-side hashing and encrypted transport, this application transmits credentials unencrypted. Modern platforms (banking, social media) employ TLS enforcement, client-side encryption, rate limiting, CAPTCHA, and generic error messages—none of which were present here.

### 3. Username Enumeration Attack

Sent login request to Burp Intruder and configured a Sniper attack targeting the username parameter. Deployed a wordlist containing common usernames derived from typical corporate naming conventions (firstname.lastname, firstnamelastname, etc.).
<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/cb6a32d0-43cd-403b-ae65-87a16bdef3a2" />

*Intruder attack results - username `azureuser` identified by response length anomaly (3,078 bytes vs 3,089 bytes)*

**Attack Results:**

| Username Type | Response Length | Error Message |
|---------------|-----------------|---------------|
| Invalid | 3,089 bytes | "Invalid username" |
| Valid | 3,078 bytes | "Incorrect password" |

**Critical Discovery:** Username `azureuser` returned response length of 3,078 bytes with error message "Incorrect password", confirming username validity. The 11-byte difference provided clear indication of different server-side code paths.

**Technical Analysis:** The application uses distinct error messages for invalid usernames vs incorrect passwords. This information leakage reduces attack complexity from O(n×m) to O(n)+O(m), transforming a 1-billion-attempt attack into a 110,000-attempt attack.

### 4. Targeted Password Brute-Force

With valid username confirmed, reconfigured Intruder for password-only attack. Fixed username to `azureuser` and loaded common password wordlist (top 10,000 passwords from breach databases).
<img width="1920" height="983" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/b94973a8-4b14-417f-82ea-2d9e32cf58e7" />

*Password brute-force successful - credentials `azureuser:123456789` identified via 302 redirect*

**Attack Results:**

| Password | Status Code | Response | Result |
|----------|-------------|----------|---------|
| test123 | 200 | Login page | Failed |
| admin | 200 | Login page   | Failed |
| 123456789 | 302 | Redirect to /my-account | Success |

**Discovery:** Password `123456789` triggered 302 redirect to `/my-account`, indicating successful authentication. This sequential numeric password is ranked in top 10 most common passwords globally and would be cracked in milliseconds by modern GPUs.

### 5. Account Takeover Validation

Validated credentials using Burp Repeater before browser-based access. Confirmed 302 response with session cookie issuance. Successfully authenticated via browser interface, gaining full access to victim account.
<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/7d72d6d5-bccb-4cbf-a9a5-bf4852f4b0e7" />

*Credentials validated in Repeater (left) | Victim account compromised (right) - email modification capability exposed*

**Compromise Confirmed:**
- Full access to account dashboard
- Email modification functionality available
- Ability to change account email enables password reset takeover
- Complete authorization to perform privileged actions

**Lab solved successfully.**

## Technical Impact

**Severity: High (CVSS 8.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Vulnerability Chain:**
1. **CWE-204:** Information leakage via response discrepancies enables username enumeration
2. **CWE-307:** Lack of rate limiting permits unlimited brute-force attempts
3. **CWE-521:** Weak password policy accepts predictable passwords
4. **CWE-319:** Plaintext credential transmission exposes data in transit

**Attack Complexity Reduction:**

| Scenario | Attempts Required | Time (10 req/sec) |
|----------|-------------------|-------------------|
| No enumeration | 1,000,000,000 | 3.2 years |
| With enumeration | 10,000 | 16 minutes |
| Reduction | 99.999% | |

**Real-World Impact:**
- **Financial Services:** Unauthorized transactions, account draining
- **Healthcare:** PHI exposure, HIPAA violations, medical record tampering
- **E-Commerce:** Fraudulent purchases, stored payment method exploitation
- **Corporate Systems:** Data exfiltration, lateral movement, privilege escalation

### Defense-in-Depth

- **WAF Deployment:** Rate limiting, bot detection, geo-blocking
- **SIEM Integration:** Alert on multiple failed attempts from single IP/username
- **Session Management:** Short timeouts, secure cookies (HttpOnly, Secure, SameSite)
- **Monitoring:** Track authentication anomalies, impossible travel, distributed attacks

## Key Takeaways

**Penetration Testing Skills:**
- Methodical reconnaissance and traffic analysis
- Response pattern identification and exploitation
- Automated attack orchestration with Burp Suite
- Credential validation and impact demonstration
- Clear documentation of attack chain and business risk

**Security Insights:**
- Information leakage (even 11 bytes) enables critical exploitation
- Rate limiting is non-negotiable for authentication endpoints
- Defense-in-depth required—single controls insufficient
- Attack surface expands exponentially with each missing control

**Professional Application:**
- Comprehensive vulnerability assessment methodology
- Automated tooling for efficient testing
- Risk quantification with CVSS scoring
- Actionable remediation guidance for development teams

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. NIST SP 800-63B - Digital Identity Guidelines
4. CWE-204, CWE-307, CWE-521 - Common Weakness Enumeration
5. OWASP Authentication Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. Unauthorized access to computer systems violates applicable laws. All techniques were performed in a controlled lab environment with explicit permission.
