# Lab 12 — Blind SQL Injection with Conditional Responses

---

## Lab Details

| Field | Details |
|-------|---------|
| **Source** | PortSwigger Web Security Academy |
| **Topic** | SQL Injection |
| **Difficulty** | Practitioner |
| **Vulnerability** | Blind SQL Injection — Boolean-Based Conditional Response Extraction |
| **CWE** | CWE-89: Improper Neutralization of Special Elements in SQL Commands |
| **Risk Level** |  Critical |
| **Completed** | 12 April 2026 |

---

## Executive Summary

A blind SQL injection vulnerability was identified in the `TrackingId` cookie parameter. The application produced no direct data output — instead, the presence or absence of a "Welcome back" message acted as a boolean oracle, revealing whether an injected condition evaluated to true or false. Through systematic enumeration of table existence, user existence, password length, and individual password characters, the administrator's full 20-character password was extracted entirely through inference. Burp Intruder's Cluster Bomb attack automated the character-by-character enumeration across all 20 positions simultaneously, assembling the complete credential set without a single byte of data ever being directly reflected in any response.

---

## Objective

Exploit a blind SQL injection vulnerability in the `TrackingId` cookie to enumerate the administrator's password character by character using boolean conditional response behaviour, then authenticate as administrator to complete the lab.

---

## Testing Setup

| Component | Details |
|-----------|---------|
| **Tool** | Burp Suite Professional (Licensed to Gurpreet Singh) |
| **Target** | PortSwigger Web Security Academy lab instance |
| **Scope** | `TrackingId` cookie parameter |
| **Method** | Manual boolean inference via Burp Repeater + Intruder |

---

## Exploitation Walkthrough

### Phase 1 — Initial Reconnaissance

With Burp Suite Professional proxying all traffic, the application was loaded in its baseline state. The page showed no "Welcome back" message between the Home and My Account navigation links — establishing the unauthenticated baseline response against which all future injections would be measured.


<img width="1920" height="983" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/07926ef0-cd00-409c-a1c3-5a39d305773e" />


*Baseline application state — "Welcome back" absent. Burp Suite Professional open and configured on the right.*

Browsing the application populated Burp's HTTP history. A `TrackingId` cookie was visible in every captured request — a parameter being passed to the backend with no visible reflection in the response, marking it as a candidate for blind injection testing.

<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/9eb707b7-6b32-4402-a4b8-fb3dfdd3c46f" />


*"Welcome back" message visible between Home and My Account when a valid TrackingId is present. Burp HTTP history showing the TrackingId cookie captured by the proxy — this conditional rendering confirmed the application was querying the cookie value against the database and altering its output based on the result.*

---

### Phase 2 — Boolean Oracle Confirmation

The request was sent to Burp Repeater. Two opposing boolean conditions were injected to verify the "Welcome back" message could be controlled through SQL logic:

**True condition:**
```sql
' AND '1'='1--
```


<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/b43922f7-4fed-43b3-a83c-3bf1c4f04af8" />


*True condition injected — "Welcome back" present in the response. The application evaluated the injected SQL and returned a positive result.*

**False condition:**
```sql
' AND '1'='0--
```


<img width="1920" height="983" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/d43586f6-c45d-44a6-bf0b-ae622505ee88" />

*False condition injected — "Welcome back" absent. The application correctly suppressed the message when the condition evaluated to false.*

The oracle was confirmed with complete reliability. Every subsequent injection would be interpreted through this single binary signal:

| Injected Condition | Evaluates To | "Welcome back" |
|-------------------|-------------|----------------|
| True | Query returns rows |  Visible |
| False | Query returns no rows |  Absent |

---

### Phase 3 — Table Existence Confirmation

With the oracle established, enumeration began by verifying the `users` table existed in the database:

```sql
' AND (SELECT 'x' FROM users LIMIT 1)='x'--
```


<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/a5355ff0-4218-4965-9efa-437f49d27b57" />

*"Welcome back" returned — the subquery successfully selected from `users`, confirming the table exists. A non-existent table would have caused the condition to fail and suppressed the message.*

---

### Phase 4 — Administrator User Confirmation

With the `users` table confirmed, the next step was verifying the `administrator` account existed as a row within it:

```sql
' AND (SELECT 'username' FROM users WHERE username='administrator')='administrator'--
```


<img width="1920" height="983" alt="LAB1_ss6 1" src="https://github.com/user-attachments/assets/1b0418fc-1a88-4f67-9c52-48fc7c7f8ed4" />

*"Welcome back" present — the WHERE clause matched a row, confirming an account with username "administrator" exists in the users table.*

To validate the oracle's negative response with equal confidence, a deliberately false username was tested:

```sql
' AND (SELECT 'username' FROM users WHERE username='random')='random'--
```


<img width="1920" height="983" alt="LAB1_ss7 1" src="https://github.com/user-attachments/assets/1ac7cb72-79ea-4448-b4cf-e3bad37ba123" />

*"Welcome back" absent — no user named "random" exists. The oracle responded correctly to both positive and negative conditions, confirming its reliability for all further enumeration.*

---

### Phase 5 — Password Length Enumeration

With the administrator account confirmed, the password length was bounded using `LENGTH()` comparisons:

**Lower bound:**
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>1)='a'--
```


<img width="1920" height="983" alt="LAB1_ss7 2" src="https://github.com/user-attachments/assets/1a42a118-2285-4d10-8f95-9d1e67046114" />

*"Welcome back" returned — password is longer than 1 character.*

**Upper bound:**
```sql
' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>20)='a'--
```


<img width="1920" height="983" alt="LAB1_ss7 3" src="https://github.com/user-attachments/assets/341cacaa-1671-43a7-93bb-41b9a9730c50" />

*"Welcome back" absent — password is not greater than 20 characters, placing the length at 20 or fewer.*

To pinpoint the exact value, the request was sent to Burp Intruder with a numeric payload cycling through 1–20 against the `LENGTH(password)>§n§` position:


<img width="1920" height="983" alt="LAB1_ss7 4" src="https://github.com/user-attachments/assets/b6122261-5c91-4aa6-83ff-580917a23f2f" />

*Intruder results — positions 1 through 19 all returned consistent response lengths (~11,600 bytes, "Welcome back" present). Position 20 returned a shorter response (~11,539 bytes, no "Welcome back") — confirming the password is exactly **20 characters long**. The response length difference provided a secondary confirmation signal alongside the message text.*

---

### Phase 6 — Character-by-Character Extraction

#### Single Character Validation (Pitchfork)

With the password length fixed at 20 characters, individual characters were extracted using `SUBSTRING()`. The first character was isolated with a single-position Pitchfork attack to validate the payload before scaling:

```sql
TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§'--
```

<img width="1920" height="983" alt="LAB1_ss8 1" src="https://github.com/user-attachments/assets/cd7268bf-7ccb-435f-92ef-f0e175198db3" />

*Intruder configured with the SUBSTRING payload — position 1 fixed, character value set as the payload cycling through alphanumeric characters.*


<img width="1920" height="983" alt="LAB1_ss8 2" src="https://github.com/user-attachments/assets/f945f31e-3930-4e20-8ff7-bd5e95a226b7" />

*Character `7` returned a distinct response length and "Welcome back" in the response — first character of the password confirmed as `7`. Payload structure validated.*

#### Full Password Extraction (Cluster Bomb)

With the single-character approach confirmed, the attack was scaled to extract all 20 characters simultaneously. A **Cluster Bomb** attack was required — unlike Pitchfork which steps through payload lists in parallel, Cluster Bomb iterates every combination of two independent payload sets, necessary here because both the character position and the character value needed to vary freely across the full attack:

- **Payload position 1** — `SUBSTRING(password,§1§,1)` — numeric values 1–20
- **Payload position 2** — `='§a§'` — full alphanumeric brute-force character list


<img width="1920" height="983" alt="LAB1_ss8 3" src="https://github.com/user-attachments/assets/3ef9a88e-2ee8-4fb1-a7ed-f97355495bc3" />

*Cluster Bomb attack configured — two independent payload positions set. Pitchfork was sufficient for single-position validation; Cluster Bomb was required here to vary both character index and character value independently across all 20 positions.*


<img width="1920" height="983" alt="LAB1_ss8 4" src="https://github.com/user-attachments/assets/8584d0d0-c92c-4c7b-ab93-f430b4d63b54" />

*Payload 1 set to numbers 1–20 (character positions). Payload 2 configured as brute-force character list covering all alphanumeric values.*


<img width="1920" height="983" alt="LAB1_ss8 5" src="https://github.com/user-attachments/assets/5822fd4c-9f3d-42c6-a6cc-d6caf3405c5d" />

*Cluster Bomb results filtered by "Welcome back" — 20 matching requests highlighted, each corresponding to one confirmed character position. All 20 successful responses returned the longer response length (~11,600 bytes). Reading the confirmed characters from position 1 through 20 produced the complete administrator password.*

---

### Phase 7 — Account Takeover

The 20-character password assembled from the Intruder results was submitted alongside the `administrator` username via the application's login page.


<img width="1920" height="983" alt="LAB1_ss8 6" src="https://github.com/user-attachments/assets/0034206d-4237-4da0-ad68-412a0efb9086" />


*Administrator credentials entered — full account access confirmed, lab solved.*

---

## Technical Impact

### Primary Vulnerability

**CWE-89: SQL Injection — Blind Boolean-Based Variant**

The `TrackingId` cookie value was incorporated directly into a backend SQL query without parameterisation. No data was ever reflected in the application's response — the only observable difference between a true and false injected condition was the presence or absence of a single UI element. That single bit of information was sufficient to enumerate the entire database through binary questioning: table existence, user existence, password length, and every individual password character were all confirmed through the same oracle, one condition at a time.

---

### Attack Chain

```
Baseline Response Captured (No "Welcome back")
                ↓
TrackingId Cookie Identified in Burp HTTP History
                ↓
Boolean Oracle Confirmed
(1=1 → Welcome back visible / 1=0 → Message absent)
                ↓
Table Existence Verified
(SELECT 'x' FROM users LIMIT 1)
                ↓
Administrator User Confirmed
(WHERE username='administrator' → True)
                ↓
False Condition Cross-Validated
(WHERE username='random' → False)
                ↓
Password Length Lower Bound (LENGTH > 1 → True)
                ↓
Password Length Upper Bound (LENGTH > 20 → False)
                ↓
Exact Length Pinpointed via Intruder
(Position 20 → Shorter Response → Password = 20 chars)
                ↓
First Character Extracted via Pitchfork
(SUBSTRING(password,1,1) = '7' → Confirmed)
                ↓
All 20 Characters Extracted via Cluster Bomb
(Positions 1–20 × Full Character Set)
                ↓
20 Matching Responses Filtered by "Welcome back"
                ↓
Complete Password Assembled
                ↓
Administrator Account Takeover
```

---

### CVSS Score

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N
Base Score: 9.1 — Critical
```

| Metric | Value |
|--------|-------|
| Attack Vector | Network |
| Attack Complexity | Low |
| Privileges Required | None |
| User Interaction | None |
| Confidentiality Impact | High |
| Integrity Impact | High |
| Availability Impact | None |

---

### Blind vs Standard SQL Injection

| Aspect | Standard SQLi | Blind Boolean SQLi |
|--------|--------------|-------------------|
| **Data Output** | Directly reflected in response | Never visible — inferred only |
| **Oracle** | Error messages or UNION output | Conditional application behaviour |
| **Extraction Speed** | Fast — full rows at once | Slow — one bit per request |
| **Detection Footprint** | High — obvious anomalies | Low — requests appear routine |
| **Tooling Required** | Repeater sufficient | Intruder automation essential |

---

### Real-World Impact

| Sector | Consequence |
|--------|-------------|
| **E-Commerce** | Customer credential enumeration through session cookie injection |
| **Healthcare** | PHI extracted character by character — HIPAA violation exposure |
| **Banking** | Account credential exfiltration without triggering error-based alerts |
| **Enterprise SaaS** | Privilege escalation through admin credential enumeration via API tokens |

---

### Why Blind SQLi is Particularly Dangerous in Production

Boolean-based blind injection generates requests that appear entirely normal to WAFs and logging systems. Each individual request looks like a routine cookie validation check. There are no error messages, no anomalous response bodies, no UNION signatures — only marginal differences in response length or a single toggled UI element. The malicious intent only becomes visible when the full sequence of hundreds of requests is analysed together, making detection significantly harder and dwell time considerably longer than with conventional injection techniques.

---

## Key Takeaways

The entire attack rested on a single observable signal — one UI element toggling on or off. The methodology was to extract maximum information from minimum feedback: table structure, user existence, password length, and every character of the credential were all derived from that same binary oracle, applied systematically at each layer before moving to the next.

The Pitchfork-to-Cluster Bomb progression reflected the same discipline. A single character was confirmed manually before the full 20-position attack was launched — validating the payload structure before committing to a large automated run. That approach keeps the attack clean, reduces noise, and means results can be trusted without ambiguity when they come back.

---

## References

| Resource | Link |
|----------|------|
| OWASP SQL Injection | https://owasp.org/www-community/attacks/SQL_Injection |
| CWE-89 | https://cwe.mitre.org/data/definitions/89.html |
| PortSwigger Blind SQLi | https://portswigger.net/web-security/sql-injection/blind |
| PortSwigger Intruder | https://portswigger.net/burp/documentation/desktop/tools/intruder |
| CVSS v3.1 Calculator | https://www.first.org/cvss/calculator/3.1 |

---

## Legal Notice

> **Author:** Gurpreet Singh
> **License:** [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)
> **Repository:** [Offensive-Security-Labs](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, credential theft, or database enumeration is illegal under applicable laws worldwide.

*© 2026 Gurpreet Singh — Offensive Security Labs*
