# Lab 8: 2FA Broken Logic

## Executive Summary

This lab demonstrates a critical flaw in two-factor authentication implementation where user identity verification relies solely on client-controlled parameters rather than server-side session management. By manipulating the `verify` cookie parameter and bypassing session validation, I successfully triggered 2FA code generation for victim accounts, brute-forced the 4-digit MFA code, and performed session hijacking to achieve complete account takeover. This vulnerability showcases the dangers of trusting client-side data for security-critical operations and highlights the importance of proper session management in multi-factor authentication flows.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | 2FA Broken Logic via Client-Side Trust |
| **Risk**               | Critical - Complete Authentication Bypass |
| **Completion**         | January 3, 2026 |

## Objective

Exploit broken 2FA logic that trusts client-controlled parameters to bypass multi-factor authentication, brute-force MFA codes, and achieve unauthorized access to victim accounts through session manipulation.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Intruder for MFA brute-forcing
- Burp Repeater for request manipulation
- Browser Developer Tools for cookie manipulation

**Target:** Two-factor authentication flow with client-side user verification

## Exploitation Walkthrough

### 1. Environment Configuration

Accessed the lab environment displaying "2FA broken logic" challenge. Configured Burp Suite Professional for comprehensive 2FA flow analysis.
<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/80c21788-d916-437a-8d47-09acbe635699" />

*Lab 8 environment - 2FA broken logic vulnerability with Burp Suite Professional configured*

### 2. Legitimate 2FA Flow Analysis

Authenticated with my credentials to understand the complete 2FA workflow and identify security weaknesses.

**Test Credentials:**
- Username: `wiener`
- Password: `peter`

Completed full authentication flow: login → email verification code → MFA submission → account access.
<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/fa3f2cb0-d2b2-4071-97f5-f67b4fbbe59c" />

*Complete 2FA flow captured in HTTP history (left) | Account accessed showing username and email (right)*

**Authentication Flow Captured:**
```
Step 1: POST /login
        username=wiener&password=peter
        Response: 302 → /login2

Step 2: GET /login2
        Cookie: session=[SESSION_TOKEN]; verify=wiener
        Response: MFA code page

Step 3: Email received with MFA code

Step 4: POST /login2
        mfa-code=[CODE]
        Cookie: session=[SESSION_TOKEN]; verify=wiener
        Response: 302 → /my-account

Step 5: GET /my-account
        Cookie: session=[SESSION_TOKEN]
        Response: Account dashboard
```

**Critical Discovery in POST /login2:**

Request contained:
- `mfa-code` parameter (user-submitted code)
- `session` cookie (session identifier)
- **`verify` cookie: username** ← VULNERABILITY

**Security Flaw Identified:** The `verify` cookie parameter contains the username and appears to control which account receives the MFA code. This client-controlled parameter is trusted by the server for security-critical operations.

### 3. GET Request Analysis for 2FA Page

Examined the GET request to `/login2` (MFA code entry page) to understand parameter trust relationships.
<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/de54b60b-0115-40f7-889c-ad3cd6969d47" />

*GET /login2 request showing session cookie and verify=wiener parameter*

**Request Structure:**
```http
GET /login2 HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: session=[SESSION_TOKEN]; verify=wiener
```

**Hypothesis:** If the server trusts the `verify` cookie to determine which user is authenticating, modifying this parameter might trigger MFA code generation for arbitrary accounts without password authentication.

### 4. Session Manipulation Testing

Sent GET /login2 request to Burp Repeater to test parameter manipulation and session validation.

**Attack Test:**
- Modified `verify` cookie from `wiener` to `carlos` (victim username)
- Removed or used existing session token

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/bbe8fdda-2c8c-4972-805e-02dd8882aafa" />

*Request manipulation in Repeater - verify cookie changed to carlos, session token removed (left) | Successful response loading MFA page for victim (right)*

**Modified Request:**
```http
GET /login2 HTTP/1.1
Cookie: verify=carlos
```
**Response:** 200 OK - MFA code entry page displayed for `carlos`

**Critical Vulnerability Confirmed:**

The application:
1. Accepts GET request to `/login2` without valid session
2. Trusts `verify` cookie parameter to determine target user
3. Generates and sends MFA code to `carlos`'s email
4. Displays MFA entry page without password verification

**Impact:** Attacker can trigger MFA code generation for any user without knowing their password by simply manipulating the `verify` cookie parameter.

### 5. MFA Code Brute-Force Attack Setup

With ability to trigger victim's MFA code generation, proceeded to brute-force the 4-digit code.

**Attack Preparation:**
1. Authenticated with my own credentials to reach MFA page
2. Captured POST /login2 request (MFA code submission)
3. Modified request for victim account targeting

**Intruder Configuration:**
- **Target Request:** POST /login2
- **Attack Type:** Sniper
- **Payload Position:** `mfa-code` parameter
- **Modified Parameters:**
  - `verify=carlos` (victim username)
  - Session cookie removed (force server to use verify parameter)
- **Payload:** 0000-9999 (all 4-digit combinations)

<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/9caf7bf4-bf4c-4eec-a229-c6bb42518ced" />

*MFA brute-force attack in progress - Valid code 1940 identified with 302 status (highlighted in green)*

**Attack Request:**
```http
POST /login2 HTTP/1.1
Cookie: verify=carlos

mfa-code=§0000§
```

**Attack Results:**

| MFA Code | Status Code | Response | Result |
|----------|-------------|----------|---------|
| 0000 | 200 | Invalid code | Failed |
| 0001 | 200 | Invalid code | Failed |
| ... | ... | ... | ... |
| 1940 | 302 | Redirect to /my-account | **Success** |
| ... | ... | ... | ... |
| 9999 | 200 | Invalid code | Failed |

**Valid MFA Code Discovered:** `1940`

**302 Response Analysis:**

The successful response contained:
```http
HTTP/1.1 302 Found
Location: /my-account
Set-Cookie: session=[CARLOS_SESSION_TOKEN]; HttpOnly
```

**Critical Data Extracted:**
- Valid MFA code: `1940`
- **Carlos's session token** from Set-Cookie header

This session token represents a fully authenticated session for the victim account.

### 6. Session Hijacking via Cookie Replacement

With victim's valid session token obtained, performed browser cookie manipulation to hijack the authenticated session.

**Attack Steps:**

**Step 1:** Accessed my own account to have a baseline session
**Step 2:** Opened Browser Developer Tools → Storage/Application → Cookies
**Step 3:** Located session cookies for the application
<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/5c298567-bc39-4550-b561-d61d17ab3ee4" />

*Cookie manipulation in Browser DevTools - Carlos's session token replacing my session (left) | Victim account fully accessed (right)*

**Original Cookie State:**
```
Name: session
Value: [MY_SESSION_TOKEN]

Name: verify  
Value: wiener
```

**Modified Cookie State:**
```
Name: session
Value: [CARLOS_SESSION_TOKEN] ← Replaced with stolen token

Name: verify
Value: carlos ← Changed to victim username (or deleted)
```

**Step 4:** Navigated to `/my-account` by modifying URL
- Changed from: `/my-account?verify=wiener`
- Changed to: `/my-account`

**Result:** Browser sent request with Carlos's session token, granting full access to victim account.

**Complete Account Access Achieved:**
- ✅ Victim's MFA bypassed through parameter manipulation
- ✅ 4-digit MFA code brute-forced (1940)
- ✅ Valid session token extracted from response
- ✅ Session hijacking successful via cookie replacement
- ✅ Full control over Carlos's account
- ✅ Username, email, and modification capabilities exposed

<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/03ae2577-27de-4a72-95d1-82a746325891" />

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**1. CWE-565: Reliance on Cookies without Validation and Integrity Checking**
- Application trusts client-controlled `verify` cookie for user identification
- No server-side session binding to authenticated user
- Cookie manipulation enables arbitrary account targeting

**2. CWE-640: Weak Password Recovery Mechanism**
- MFA code generation triggered without password verification
- 4-digit codes susceptible to brute-force (10,000 combinations)
- No rate limiting on MFA code attempts

**3. CWE-384: Session Fixation**
- Session tokens not properly bound to authentication context
- Stolen session tokens usable across different accounts
- No session invalidation after suspicious activity

**Vulnerability Chain:**
```
Client-Side Parameter Trust
        ↓
Manipulate verify=carlos Cookie
        ↓
Trigger MFA Code Generation (no password)
        ↓
Brute-Force 4-Digit MFA Code
        ↓
Valid Code: 1940
        ↓
Extract Session Token from Response
        ↓
Replace Browser Session Cookie
        ↓
Complete Account Takeover
```

**Broken 2FA Implementation:**
```python
# VULNERABLE CODE
@app.route('/login2', methods=['GET'])
def get_mfa_page():
    # FLAW 1: Trusts client-controlled cookie
    username = request.cookies.get('verify')
    
    # FLAW 2: No password verification required
    # FLAW 3: No session validation
    send_mfa_code(username)
    return render_mfa_page()

@app.route('/login2', methods=['POST'])
def verify_mfa():
    # FLAW 4: Still trusts verify cookie
    username = request.cookies.get('verify')
    mfa_code = request.form.get('mfa-code')
    
    # FLAW 5: No rate limiting on attempts
    if verify_mfa_code(username, mfa_code):
        # FLAW 6: Creates session without proper binding
        session_token = create_session(username)
        return redirect('/my-account')
    
    return "Invalid code"
```

**Secure Implementation:**
```python
# SECURE CODE
@app.route('/login2', methods=['GET'])
def get_mfa_page():
    # Verify user already authenticated with password
    if not session.get('password_verified'):
        return redirect('/login')
    
    # Use server-side session, never trust cookies
    username = session.get('username')
    
    # Send MFA only to authenticated user
    send_mfa_code(username)
    return render_mfa_page()

@app.route('/login2', methods=['POST'])
@rate_limit('5/minute')  # Rate limiting
def verify_mfa():
    # Verify password authentication first
    if not session.get('password_verified'):
        return redirect('/login'), 401
    
    # Use server-side session data
    username = session.get('username')
    mfa_code = request.form.get('mfa-code')
    
    # Time-limited code validation
    if not is_code_valid_and_not_expired(username, mfa_code):
        session['mfa_attempts'] += 1
        if session['mfa_attempts'] >= 3:
            session.clear()
            return "Too many attempts", 429
        return "Invalid code", 401
    
    # Bind session to user after full verification
    session['mfa_verified'] = True
    session['fully_authenticated'] = True
    
    return redirect('/my-account')
```

**Why This Attack Succeeds:**

1. **Client-Side Trust:** Server trusts `verify` cookie to determine user identity
2. **No Password Requirement:** MFA page accessible without password verification
3. **Weak MFA Codes:** 4-digit codes = 10,000 combinations (easily brute-forced)
4. **No Rate Limiting:** Unlimited MFA attempts enable brute-force
5. **Session Token Exposure:** Valid session tokens returned in responses
6. **No Session Binding:** Session tokens not bound to original authentication context

**Real-World Attack Scenarios:**

| Target | Impact |
|--------|--------|
| **Banking** | Complete account takeover, fund transfers |
| **Email Services** | Full mailbox access, password resets for other services |
| **Corporate Systems** | Data exfiltration, privilege escalation |
| **Healthcare** | PHI exposure, record manipulation |
| **E-Commerce** | Fraudulent purchases, customer data theft |

**Attack Complexity:** Moderate - requires understanding of 2FA flows and session management, but execution is straightforward with proper tools.

## Key Takeaways

**Penetration Testing Methodology:**
- Comprehensive analysis of multi-stage authentication flows
- Client-side parameter manipulation testing
- Session token extraction and hijacking techniques
- Brute-force attacks against weak MFA implementations
- Cookie manipulation for session impersonation

**Critical Security Insights:**

**1. Never Trust Client-Side Data for Security Decisions:**
Server-side sessions must maintain authentication state:
- User identity should come from server-side session, never cookies
- Client-controlled parameters cannot be trusted for security operations
- All security decisions must be based on server-validated data

**2. MFA Must Build on Password Authentication:**
2FA is a second factor, not a replacement:
- Password verification must complete before MFA
- MFA page should only be accessible to password-authenticated users
- Server must track authentication progress server-side

**3. MFA Codes Require Protection:**
4-digit codes are insufficient without additional controls:
- Minimum 6-digit codes recommended (1,000,000 combinations)
- Rate limiting mandatory (3-5 attempts maximum)
- Time-limited validity (30-60 seconds)
- Account lockout after failed attempts

**4. Session Management is Critical:**
Proper session security requires:
- Session binding to authentication context
- Server-side session state tracking
- Session invalidation on suspicious activity
- No session token exposure in responses
- HttpOnly, Secure, SameSite cookie flags

**5. Defense-in-Depth for 2FA:**
Multiple layers prevent bypass:
- Server-side authentication state tracking
- Rate limiting on MFA attempts
- Strong MFA codes (6+ digits or TOTP)
- Session binding and validation
- Anomaly detection and monitoring

**Professional Skills Demonstrated:**
- Advanced authentication flow analysis
- Session management vulnerability identification
- Multi-stage attack orchestration
- Client-side security control bypass
- Clear documentation of complex exploitation paths

## References

1. PortSwigger Web Security Academy - Authentication Vulnerabilities
2. OWASP Top 10 (2021) - A07: Identification and Authentication Failures
3. CWE-565 - Reliance on Cookies without Validation
4. CWE-384 - Session Fixation
5. NIST SP 800-63B - Digital Identity Guidelines
6. OWASP Session Management Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
