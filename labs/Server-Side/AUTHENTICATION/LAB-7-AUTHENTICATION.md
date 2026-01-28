# Lab 7: Username Enumeration via Account Lock

## Executive Summary

This lab demonstrates username enumeration through account lockout behavior differences. The application implements brute-force protection that locks accounts after multiple failed attempts, but this security control inadvertently reveals valid usernames through distinct error messages and response patterns. By systematically triggering account lockouts, I identified valid usernames, then exploited response variations to discover passwords despite lockout mechanisms. This showcases how defensive security controls can become information disclosure vectors when improperly implemented.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Username Enumeration via Account Lock Behavior |
| **Risk**               | High - Authentication Bypass via Lockout Exploitation |
| **Completion**         | January 3, 2026 |

## Objective

Exploit account lockout mechanisms to enumerate valid usernames through behavioral differences, then leverage response variations to identify correct passwords despite active brute-force protections.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Intruder with Cluster Bomb attack
- Burp Repeater for validation testing
- Burp Proxy for traffic interception

**Target:** Login functionality with account lockout protection

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed the lab environment displaying "Username enumeration via account lock" challenge. Configured Burp Suite Professional for lockout behavior analysis.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/e95bd51b-fe4c-4fa5-b4e7-ec306cf35365" />

*Lab 7 environment - Account lock enumeration vulnerability with Burp Suite Professional configured*

### 2. Initial Authentication Request Capture

Submitted test credentials to capture authentication request structure and analyze server behavior.

**Test Credentials:**
- Username: `carlos`
- Password: `adam%4012345`

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/adcf61bd-f1fd-4331-8d75-6e5b87105a35" />

*Invalid credentials captured in HTTP history - POST request showing username=carlos&password=adam%4012345*

**Request Structure:**
```http
POST /login HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&password=adam%4012345
```
### 3. Brute-Force Protection Testing

Sent captured request to Burp Repeater to test for rate limiting and account lockout mechanisms.

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/0bca7333-1446-47b3-be18-3f8d64285085" />

*Multiple requests sent in Repeater - no immediate IP-based rate limiting detected*

**Testing Results:**
- Multiple rapid requests accepted without IP blocking
- No rate limiting on request frequency observed
- Response times consistent across attempts
- Error messages consistent for invalid credentials

**Initial Assessment:** No obvious IP-based brute-force protection, but further testing required to identify account-level protections.

### 4. Username Enumeration via Lockout Triggering

**Attack Strategy:** Use Cluster Bomb attack to repeatedly attempt authentication with each username, triggering account lockout for valid accounts.

**Intruder Configuration:**
- **Attack Type:** Cluster Bomb (multiple payload sets)
- **Payload Position 1:** Username field
- **Payload Position 2:** Null payload (repetition mechanism)

**Payload Sets:**
- **Payload Set 1:** Username wordlist (common usernames)
- **Payload Set 2:** Null payloads with 5 iterations

**Purpose:** Attempt each username exactly 5 times to trigger account lockout protection.
<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/805bfd03-f78b-481a-a4af-6641fcb6d844" />

*Cluster Bomb attack results - Username "adam" highlighted in green showing different response pattern after multiple attempts*

**Attack Pattern:**
```
username=admin, iteration=1 → "Invalid username or password"
username=admin, iteration=2 → "Invalid username or password"
username=admin, iteration=3 → "Invalid username or password"
username=admin, iteration=4 → "Invalid username or password"
username=admin, iteration=5 → "Invalid username or password"

username=adam, iteration=1 → "Invalid username or password"
username=adam, iteration=2 → "Invalid username or password"
username=adam, iteration=3 → "Invalid username or password"
username=adam, iteration=4 → "Invalid username or password"
username=adam, iteration=5 → "You have made too many incorrect login attempts"
```

**Critical Discovery:**

| Username Type | Response After 5 Attempts | Indicator |
|---------------|---------------------------|-----------|
| Invalid | "Invalid username or password" | No lockout |
| Valid | "You have made too many incorrect login attempts" | Account locked |

**Valid Username Identified:** `adam`

**Technical Explanation:**

The application implements account lockout only for existing accounts:
```python
# VULNERABLE LOGIC
def authenticate(username, password):
    user = database.find_user(username)
    
    if user is None:
        return "Invalid username or password"  # No lockout for non-existent users
    
    if user.is_locked():
        return "You have made too many incorrect login attempts"
    
    if not verify_password(password, user.password_hash):
        user.increment_failed_attempts()
        
        if user.failed_attempts >= 5:
            user.lock_account()
            return "You have made too many incorrect login attempts"
        
        return "Invalid username or password"
    
    return success()
```

**Vulnerability:** Account lockout behavior reveals username validity. Invalid usernames never trigger lockout messages, while valid usernames display distinct lockout errors after threshold.

### 5. Password Enumeration Against Locked Account

With valid username confirmed, conducted password brute-force attack to identify correct credentials.

**Attack Configuration:**
- **Attack Type:** Sniper
- **Username:** `adam` (fixed - confirmed valid)
- **Target Parameter:** Password
- **Payload:** Common password wordlist

<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/15d41a2b-4165-4791-b584-47bf68534af0" />

*Password brute-force attack - All attempts show 200 status, but password "ginger" shows different error message (highlighted in green)*

**Attack Results:**

| Password | Status Code | Response Message | Analysis |
|----------|-------------|------------------|----------|
| 123456 | 200 | "Invalid username or password" | Wrong password |
| password | 200 | "Invalid username or password" | Wrong password |
| admin | 200 | "Invalid username or password" | Wrong password |
| ginger | 200 | **(No error message)** | **Correct password!** |
| letmein | 200 | "Invalid username or password" | Wrong password |

**Critical Finding:**

The correct password returns:
- **Status Code:** 200 (not 302 redirect as typical)
- **Response:** No "Invalid username or password" error
- **Behavior:** Different response content despite same status code

**Password Identified:** `ginger`

**Why This Works:**

Even with account locked, the correct password produces a different response:
- **Wrong password:** "Invalid username or password" error displayed
- **Right password:** Account locked message or different response (no invalid credentials error)

The application still validates the password even when the account is locked, leaking information through response differences.

### 6. Credential Validation & Account Access

Validated discovered credentials using Burp Repeater before browser authentication.
<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/b60b88c1-0d04-4dee-8f6f-d811afdf293c" />

*Credentials validated - 302 redirect in Repeater (left) | Victim account accessed showing username and email modification capability (right)*

**Validation Request:**
```http
POST /login HTTP/1.1

username=adam&password=ginger
```

**Response:** 302 Found - Redirect to `/my-account`

**Account Access Achieved:**
- ✅ Valid username enumerated via lockout behavior
- ✅ Password discovered through response analysis
- ✅ Full authentication successful
- ✅ Account dashboard accessible
- ✅ Email modification functionality available
- ✅ Complete control over victim account

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: High (CVSS 8.1)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerability:**

**CWE-204: Observable Response Discrepancy**
- Account lockout behavior differs between valid and invalid usernames
- Lockout messages only displayed for existing accounts
- Password validation occurs even on locked accounts, leaking information
- Response content variations enable credential discovery

**Vulnerability Chain:**
```
Observable Lockout Behavior
        ↓
Username Enumeration (via lockout message)
        ↓
Valid Username: "adam"
        ↓
Password Brute-Force on Locked Account
        ↓
Response Variation Detection
        ↓
Correct Password: "ginger"
        ↓
Complete Account Takeover
```

**Flawed Security Logic:**
```python
# VULNERABLE IMPLEMENTATION
def authenticate(username, password):
    user = find_user(username)
    
    # FLAW 1: Different behavior for invalid usernames
    if user is None:
        return "Invalid username or password"  # Never locks
    
    # FLAW 2: Lockout message reveals username validity
    if user.is_locked():
        return "You have made too many incorrect login attempts"
    
    # FLAW 3: Password validation on locked accounts
    if verify_password(password, user.password_hash):
        return success()  # Different response even when locked
    
    user.increment_failed_attempts()
    return "Invalid username or password"
```

**Secure Implementation:**
```python
# SECURE IMPLEMENTATION
def authenticate(username, password):
    # Always perform same operations
    user = find_user(username)
    
    # Use dummy user for invalid usernames
    if user is None:
        user = create_dummy_user()
    
    # Check lockout without revealing username validity
    if user.is_locked():
        # Generic message - same for all scenarios
        return "Invalid username or password", 401
    
    # Constant-time password verification
    password_valid = verify_password_constant_time(password, user.password_hash)
    
    if user.is_real and password_valid:
        return success()
    
    if user.is_real:
        user.increment_failed_attempts()
    
    # Always return identical response
    return "Invalid username or password", 401
```

**Why Account Lockout Leaks Information:**

1. **Differential Treatment:** Valid and invalid usernames handled differently
2. **Distinct Messages:** Lockout messages only appear for real accounts
3. **Enumeration Shortcut:** Attackers can map entire user database
4. **Password Validation Leak:** Even locked accounts validate passwords differently

**Real-World Attack Scenarios:**

| Target | Impact |
|--------|--------|
| **Corporate Email** | Employee account enumeration for phishing |
| **Banking Systems** | Customer account identification |
| **Healthcare Portals** | Patient account discovery |
| **E-Commerce** | Customer database mapping |
| **Social Media** | User existence verification |

**Attack Complexity:** Moderate - requires understanding of lockout mechanisms and response analysis, but execution is straightforward with Burp Suite.

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic testing of defensive mechanisms for information leakage
- Cluster Bomb attacks for lockout triggering
- Response content analysis beyond status codes
- Pattern recognition in security control behavior

**Critical Security Insights:**

**1. Security Controls Can Leak Information:**
Defensive mechanisms like account lockout can inadvertently become enumeration vectors when they:
- Produce different messages for valid vs invalid accounts
- Respond differently based on account state
- Fail to normalize responses across all scenarios

**2. Lockout Should Be Uniform:**
Proper account lockout implementation requires:
- Identical responses for all usernames (valid or not)
- No behavioral differences revealing account existence
- Same timing and message content regardless of account state

**3. Defense-in-Depth Must Be Consistent:**
Multiple security layers should not contradict each other:
- Account lockout implemented to prevent brute-force
- But lockout behavior enables username enumeration
- Defense mechanism becomes attack vector

**4. Response Normalization is Critical:**
All authentication failures must return identical responses:
- Same status code
- Same message content
- Same response length
- Same timing characteristics
- Same behavior regardless of internal state

**Professional Skills Demonstrated:**
- Analysis of security control implementation flaws
- Creative use of Cluster Bomb attack patterns
- Response content analysis techniques
- Understanding of authentication protection pitfalls
- Clear documentation of multi-stage exploitation

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-204 - Observable Response Discrepancy
4. CWE-307 - Improper Restriction of Excessive Authentication Attempts
5. OWASP Authentication Cheat Sheet
6. NIST SP 800-63B - Digital Identity Guidelines

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
