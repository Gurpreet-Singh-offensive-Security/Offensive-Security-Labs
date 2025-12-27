# Web Cache Deception - Lab Series

## Overview

This directory contains a comprehensive series of hands-on labs exploring **Web Cache Deception** vulnerabilities. These labs progress from foundational concepts to expert-level exploitation techniques, demonstrating real-world attack scenarios against misconfigured caching systems.

## What is Web Cache Deception?

Web Cache Deception is a vulnerability that occurs when there are **normalization discrepancies** between cache layers (CDNs, reverse proxies) and origin servers. Attackers exploit these differences to trick the cache into storing sensitive, authenticated content under publicly accessible cache keys.

### Attack Mechanism

1. **Attacker crafts malicious URL** using path manipulation, delimiters, or encoding
2. **Cache layer** interprets the URL as a static, cacheable resource
3. **Origin server** normalizes the URL differently, serving dynamic authenticated content
4. **Cache stores** the authenticated response as if it were public static content
5. **Attacker retrieves** cached sensitive data without authentication

### Real-World Impact

- 🔓 **Authentication bypass** - Access protected resources without credentials
- 🔑 **Credential exposure** - Extract API keys, tokens, and session data
- 👤 **Account takeover** - Steal CSRF tokens and hijack user sessions
- 📊 **Data exfiltration** - Access personal information, financial data, PII
- 🎯 **Privilege escalation** - Compromise administrative accounts

## Lab Series Structure

This series follows a **progressive difficulty model**, building from basic concepts to complex multi-stage attacks:

| Lab | Difficulty | Focus Area | Key Techniques |
|-----|------------|------------|----------------|
| [Lab 1](./lab-1-web-cache-deception.md) | Apprentice   | Static Extension Exploitation | Basic path manipulation, extension-based caching |
| [Lab 2](./lab-2-web-cache-deception.md) | Practitioner | Cache Key Vulnerabilities | Cache key flaws, origin header manipulation |
| [Lab 3](./lab-3-web-cache-deception.md) | Practitioner | Cache Rule Bypass | Delimiter discovery, path truncation |
| [Lab 4](./lab-4-web-cache-deception.md) | Practitioner | Normalization Discrepancies | URL encoding, path traversal, delimiter fuzzing |
| [Lab 5](./lab-5-web-cache-deception.md) | Expert       | Exact Match Exploitation | Multi-stage attacks, CSRF chaining, session timing |

## Learning Path

### Prerequisites
- HTTP fundamentals and caching concepts
- Burp Suite proficiency (Proxy, Repeater, Intruder)
- Understanding of authentication mechanisms
- Basic knowledge of URL encoding and normalization

### Recommended Order
1. **Start with Lab 1** - Builds foundation in cache behavior analysis
2. **Progress through Labs 2-4** - Develops systematic vulnerability discovery skills
3. **Complete Lab 5** - Demonstrates real-world attack complexity

### Skills Developed
- ✅ Cache behavior analysis and policy identification
- ✅ Systematic fuzzing and delimiter discovery
- ✅ URL normalization vulnerability detection
- ✅ Multi-stage attack chain construction
- ✅ Session timing and cache poisoning exploitation
- ✅ CSRF token extraction and reuse
- ✅ Professional security reporting and documentation

## Lab Summaries

### Lab 1: Static Extension Exploitation
**Vulnerability:** Cache serves dynamic content when static file extensions are appended  
**Technique:** Path manipulation with `.js` extension  
**Impact:** API key exposure through basic cache poisoning  
**Key Learning:** Foundation of cache/origin normalization gaps

---

### Lab 2: Cache Key Vulnerabilities
**Vulnerability:** Cache key generation excludes specific parameters or headers  
**Technique:** Origin header manipulation and cache key analysis  
**Impact:** Bypass cache keying to retrieve authenticated content  
**Key Learning:** Understanding cache key composition and exclusions

---

### Lab 3: Cache Rule Bypass via Delimiters
**Vulnerability:** Delimiter handling differences between cache and origin  
**Technique:** Systematic delimiter fuzzing with Burp Intruder  
**Impact:** Path truncation leading to sensitive data caching  
**Key Learning:** Delimiter-based normalization exploitation

---

### Lab 4: Normalization Discrepancies
**Vulnerability:** URL encoding and path traversal handling inconsistencies  
**Technique:** Encoded delimiter combinations with path traversal  
**Impact:** Cache poisoning with authenticated user data  
**Key Learning:** Advanced multi-encoding attack construction

---

### Lab 5: Exact Match Cache Rules (Expert)
**Vulnerability:** Exact match rules with delimiter normalization gaps  
**Technique:** Multi-stage attack combining cache deception, session hijacking, and CSRF  
**Impact:** Complete administrative account takeover  
**Key Learning:** Complex attack chain orchestration and timing exploitation

## Tools & Setup

### Required Tools
- **Burp Suite Professional** - Primary testing platform
  - Proxy for traffic interception
  - Repeater for request manipulation
  - Intruder for systematic fuzzing
  - CSRF PoC Generator (Lab 5)
- **Web Browser** - Session management and authentication
- **PortSwigger Academy Account** - Lab access and exploit server

### Environment Setup
1. Configure browser proxy settings (127.0.0.1:8080)
2. Install Burp Suite CA certificate
3. Create PortSwigger Academy account
4. Familiarize yourself with Burp Suite tools

## Common Attack Patterns

### Pattern 1: Extension-Based Deception
```
/sensitive-endpoint.js
/api/user-data.css
/my-account.png
```
**Cache sees:** Static file extension → Cache  
**Origin sees:** Dynamic endpoint → Serve authenticated content

### Pattern 2: Delimiter Exploitation
```
/my-account?foo.js
/api/data;static.css
/profile#/../resources/style.css
```
**Cache sees:** Static resource indicator → Cache  
**Origin sees:** Delimiter truncates/normalizes → Serve dynamic content

### Pattern 3: Path Traversal with Encoding
```
/account/../resources/static.js
/api/..%2Fresources%2Fstyle.css
/data;%2f%2e%2e%2frobots.txt
```
**Cache sees:** Literal path with static indicator → Cache  
**Origin sees:** Decoded/normalized path → Serve sensitive endpoint

### Pattern 4: Cache Key Exclusions
```
GET /api/user HTTP/1.1
X-Forwarded-Host: evil.com
Origin: https://evil.com
```
**Cache sees:** Excludes certain headers from key → Wrong cached content  
**Origin sees:** Full request with manipulated headers → Serve user-specific data

## Defense Mechanisms

### Secure Configuration Checklist

**Cache Layer:**
- ✅ Implement unified URL normalization with origin server
- ✅ Use explicit allowlists for cacheable resources (no prefix matching)
- ✅ Include authentication state in cache key generation
- ✅ Set strict Cache-Control headers on dynamic content
- ✅ Validate delimiter handling matches origin server behavior

**Origin Server:**
- ✅ Return `Cache-Control: no-store, private` on authenticated endpoints
- ✅ Implement consistent delimiter parsing
- ✅ Reject ambiguous URLs with encoded traversal sequences
- ✅ Use separate domains for static vs dynamic content

**Security Monitoring:**
- ✅ Alert on cache hits for dynamic/authenticated endpoints
- ✅ Monitor unusual URL patterns (encoded characters, delimiters)
- ✅ Log cache key generation for security audits
- ✅ Track rapid sequential access to recently cached resources

## Testing Methodology

### Step-by-Step Approach

1. **Reconnaissance**
   - Map application endpoints (authenticated vs public)
   - Identify sensitive data locations
   - Analyze cache behavior (headers, timing)

2. **Cache Policy Discovery**
   - Test basic path modifications
   - Identify cached vs non-cached resources
   - Document cache headers and TTL values

3. **Delimiter Fuzzing**
   - Use Burp Intruder with delimiter payloads
   - Test both raw and encoded characters
   - Identify normalization discrepancies

4. **Exploit Construction**
   - Combine findings into attack URL
   - Test cache poisoning with authenticated session
   - Verify unauthorized cache retrieval

5. **Impact Validation**
   - Extract sensitive data from cache
   - Document exploitability and impact
   - Test persistence and scope

## Additional Resources

### Official Documentation
- [PortSwigger Web Security Academy - Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [RFC 7234 - HTTP/1.1 Caching](https://tools.ietf.org/html/rfc7234)
- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)

### Research Papers & Talks
- Omer Gil - "Web Cache Deception Attack" (BlackHat USA 2017)
- James Kettle - "Practical Web Cache Poisoning" (BlackHat USA 2018)
- OWASP - Web Cache Deception Attack Guide

### Related Topics
- Web Cache Poisoning
- HTTP Request Smuggling
- Server-Side Request Forgery (SSRF)
- Cross-Site Request Forgery (CSRF)

## Contributing

Found an issue or have improvements? Contributions are welcome!

1. Fork the repository
2. Create a feature branch
3. Submit a pull request with detailed description

## Legal & Ethical Notice

⚠️ **CRITICAL DISCLAIMER**

All techniques documented in these labs are for:
- **Authorized security testing only**
- **Educational purposes in controlled environments**
- **Professional penetration testing with explicit permission**

**Unauthorized access to computer systems is illegal** under laws including:
- Computer Fraud and Abuse Act (CFAA) - United States
- Computer Misuse Act - United Kingdom  
- Similar legislation in other jurisdictions

The author assumes **no liability** for misuse of these techniques.

## Author & License

**Author:** Gurpreet Singh  
**Repository:** [Offensive-Security-Labs](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)  
**License:** CC BY-NC-ND 4.0 (Attribution-NonCommercial-NoDerivatives)

**Copyright © 2025 Gurpreet Singh. All rights reserved.**

---

## Lab Completion Tracker

Track your progress through the series:

- [ ] Lab 1: Static Extension Exploitation
- [ ] Lab 2: Cache Key Vulnerabilities  
- [ ] Lab 3: Cache Rule Bypass via Delimiters
- [ ] Lab 4: Normalization Discrepancies
- [ ] Lab 5: Exact Match Cache Rules (Expert)
**Good luck and happy hunting! 🎯🔒**
