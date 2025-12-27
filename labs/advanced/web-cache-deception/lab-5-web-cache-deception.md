# Lab 5: Exploiting Exact Match Cache Rules for Web Cache Deception

## Executive Summary

This lab demonstrates an advanced web cache deception vulnerability caused by exact match cache rules and delimiter-based normalization inconsistencies. By exploiting semicolon delimiter handling and URL-encoded path traversal combined with a publicly cacheable resource (robots.txt), I successfully poisoned the cache with the victim's authenticated session data. This enabled CSRF token extraction and subsequent account takeover through email modification, all without direct access to the victim's credentials.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Expert |
| **Vulnerability**      | Web Cache Deception via Exact Match Cache Rules |
| **Risk**               | Critical - Session Hijacking & Account Takeover |
| **Completion**         | December 27, 2025 |

## Objective

Exploit exact match cache rules and delimiter normalization to poison the cache with authenticated administrative content, extract the administrator's CSRF token, and perform unauthorized email modification to complete account takeover.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Exploit Server for payload delivery
- CSRF PoC Generator for attack automation
- Browser for session management

**Target:** "/my-account" endpoint containing email modification functionality with CSRF protection

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Started by intercepting traffic through Burp Suite and identifying the lab environment. Authenticated to the application and accessed the "/my-account" endpoint to understand available functionality.
<img width="1920" height="982" alt="LAB5_ss1" src="https://github.com/user-attachments/assets/c28f6590-b30d-48f7-9829-d7c6e6785cee" />

*Lab 5 environment - Expert level Web Cache Deception with Burp Suite Professional configured*

### 2. Sensitive Functionality Discovery

Explored the account management interface and identified a critical email modification feature. This functionality allows authenticated users to change their email address, which if exploited on an administrator account, would enable complete account takeover through password reset mechanisms.
<img width="1920" height="982" alt="LAB5_ss2" src="https://github.com/user-attachments/assets/b65cf37e-3ab7-4378-9cfd-abee841ccd7d" />

*Email modification functionality discovered - potential attack vector for credential takeover*

**Security Impact:** Access to this endpoint with administrative privileges would allow an attacker to:
- Modify the administrator's email to attacker-controlled address
- Initiate password reset to the new email
- Achieve full account takeover without knowing original credentials

### 3. Path Behavior Testing

Tested basic path manipulation by modifying the GET request from "/my-account" to "/my-account/abc". Received a 404 response with no caching headers, indicating strict path matching without automatic path normalization or flexible delimiter handling.
<img width="1920" height="982" alt="LAB5_ss3" src="https://github.com/user-attachments/assets/04fcda77-ef31-4dd0-ad3b-b25cb4f23e1a" />

*Strict path matching confirmed - arbitrary path extensions rejected*

**Finding:** The server enforces exact path matching, suggesting that any successful cache deception attack must work within strict normalization rules while exploiting delimiter handling discrepancies.

### 4. Delimiter Discovery via Fuzzing

Used Burp Intruder with a comprehensive delimiter payload list to systematically identify characters the server recognizes as valid delimiters. **Critical configuration:** Ensured URL encoding settings were properly configured to test both raw and encoded delimiter variations.

**Discovered delimiters returning 200 OK responses:**
- ' ; ' (semicolon)
- ' ? ' (question mark)
<img width="1920" height="982" alt="LAB5_ss4" src="https://github.com/user-attachments/assets/82609534-af43-47ed-bb28-7f24888bd089" />

*Intruder attack results showing valid delimiters (200 responses highlighted in green)*

**Technical Analysis:** The 200 status codes indicate these delimiters are recognized by the origin server as valid path terminators or query string initiators, creating potential normalization gaps with the cache layer.

### 5. Cache Rule Investigation with Delimiters

Tested cache behavior by appending discovered delimiters with static-looking filenames to the target endpoint:
- `/my-account;foo.js`
- `/my-account?foo.js`

Both requests returned responses without cache headers (`Cache-Control`, `X-Cache`), indicating that simply using delimiters with static extensions is insufficient to trigger caching policies.
<img width="1920" height="982" alt="LAB5_ss5" src="https://github.com/user-attachments/assets/705ec162-cfad-4a50-96f5-5ef26b682853" />

*Delimiter testing with static extensions - no cache headers observed*

**Insight:** The cache implementation uses more sophisticated matching rules than simple extension-based or prefix-based caching, requiring exact match or specific path patterns.

### 6. Encoded Path Traversal Testing

Tested whether the server decodes URL-encoded path traversal sequences by attempting:
`/aaa/..%2Fmy-account`

Received a 404 response, confirming the server does NOT decode the `%2F` (encoded forward slash) before path resolution.

<img width="1920" height="982" alt="LAB5_ss6" src="https://github.com/user-attachments/assets/bfa9d5ba-10d3-4bcc-95f6-dfd16a42dc60" />

*Encoded path traversal rejected - server processes encoded characters literally*

**Finding:** This behavior is critical - it means the cache and origin server likely handle encoded characters identically, eliminating encoded traversal as a direct attack vector but suggesting encoded characters can be used in cache key construction.

### 7. Static Resource Cache Analysis

Analyzed HTTP proxy history for evidence of cached static resources. Notably, even requests to the "/resources/" directory showed no consistent cache headers, suggesting the application uses exact match cache rules rather than prefix-based caching.

<img width="1920" height="982" alt="LAB5_ss7" src="https://github.com/user-attachments/assets/7847cf5c-9012-4176-ace4-80c176ee2681" />

*Proxy history analysis - no prefix-based caching observed on static directories*

**Implication:** The cache system likely maintains an allowlist of specific cacheable resources rather than caching all resources under certain paths. This requires identifying an explicitly cacheable resource to exploit.

### 8. Robots.txt Cache Discovery

Tested the commonly cacheable resource `/robots.txt`, which is typically cached by CDNs and reverse proxies to reduce load on origin servers since it's frequently requested by web crawlers and security scanners.

**First request:** Returned `X-Cache: miss` - cache attempted to store the response
**Second request:** Same path returned `X-Cache: hit` - cache successfully served stored response

<img width="1920" height="982" alt="LAB5_ss8" src="https://github.com/user-attachments/assets/3a713bd0-f86a-4bd5-a4dc-1bddb64f684b" />

*Cache behavior confirmed on robots.txt - ideal target for cache key manipulation*

**Why robots.txt?**
- Universal presence on web servers (standards compliance)
- High request frequency from crawlers and bots
- Typically considered safe/public content, thus aggressively cached
- Exact path match makes it perfect for exact match cache rule exploitation

### 9. Exploit Construction - Cache Key Collision

Combined all findings to craft the attack URL that creates a cache key collision:
```
/my-account;%2f%2e%2e%2frobots.txt
```

**How the exploit works:**

**Cache layer processing:**
1. Sees the literal string path: `/my-account;%2f%2e%2e%2frobots.txt`
2. Applies exact match rules and identifies `robots.txt` at the end
3. Determines this matches the cacheable resource rule for `/robots.txt`
4. Generates cache key based on this literal path
5. Applies caching policy (stores response)

**Origin server processing:**
1. Receives: `/my-account;%2f%2e%2e%2frobots.txt`
2. Recognizes `;` (semicolon) as a delimiter/parameter separator
3. Truncates the path at the delimiter: `/my-account`
4. Processes encoded characters literally: `%2f%2e%2e` remains encoded in the ignored portion
5. Routes to `/my-account` endpoint, serving authenticated content

**First test request:** Returned 200 OK with `X-Cache: miss`

<img width="1920" height="982" alt="LAB5_ss9 1" src="https://github.com/user-attachments/assets/628c97f5-24cc-4792-a5df-67c0272bc83d" />

**Second test request:** Same path returned 200 OK with `X-Cache: hit`
<img width="1920" height="982" alt="LAB5_ss9 2" src="https://github.com/user-attachments/assets/f43ae0b2-f5ee-461d-b84e-54b505377805" />

*Exploit path successfully cached - normalization discrepancy confirmed*

### 10. Initial Exploit Delivery Attempt

Configured the exploit server to deliver the malicious URL to the victim (administrator). The exploit URL would cause the victim's authenticated session to request the crafted path, poisoning the cache with their administrative session data.

<img width="1920" height="982" alt="LAB5_ss10" src="https://github.com/user-attachments/assets/e5179de0-aaff-40bc-9f4b-c75daa6b83eb" />

*Exploit server configured for victim delivery*

### 11. Session Timing Discovery

After delivering the exploit to the victim, immediately attempted to access the same URL to retrieve cached administrative data. However, was redirected to the login page, indicating the cached content included session-specific data or had session-binding mechanisms.

<img width="1920" height="982" alt="LAB5_ss11" src="https://github.com/user-attachments/assets/3f0d1f69-a404-4b07-8bf4-6ac739ebad10" />


*Login page redirect - cached content tied to session state*

**Critical discovery:** The cached response is tied to the session that created it, requiring the attack to be executed within the victim's active session window (approximately 30-second window based on cache behavior).

### 12. Optimized Exploit with Cache Buster

Refined the attack strategy to work within session timing constraints:

1. **Added cache buster parameter:** Modified exploit URL to `/my-account;%2f%2e%2e%2frobots.txt?dgc`
   - The query parameter `?dgc` creates a unique cache key while maintaining exploitation
   - Prevents cache reuse from previous attempts
   - Both exploit server and Burp Repeater used identical cache buster value

2. **Rapid execution sequence:**
   - Delivered exploit to victim via exploit server
   - Immediately (within 30-second window) sent identical request via Burp Repeater
   - Repeater request caught the cached administrative session data

**Response obtained:** Successfully retrieved administrator's authenticated response containing CSRF token

<img width="1920" height="982" alt="LAB5_ss12" src="https://github.com/user-attachments/assets/ff09de34-1e6c-4d39-8f96-67a262a5eaf5" />


*Administrator session data retrieved from cache - CSRF token extracted*

**Extracted data:**
- Administrator's CSRF token: `[token value visible in response]`
- Session-specific data confirming administrative privileges
- Email modification form structure and parameters

### 13. CSRF Attack Construction

With the administrator's CSRF token extracted, proceeded to craft the account takeover attack:

**Step 1: Token Replacement**
Intercepted the email change request from my authenticated session and modified it:
- **Original CSRF token:** My user's token
- **Replaced with:** Administrator's extracted CSRF token
- **Email parameter:** Changed to attacker-controlled email address

<img width="1920" height="982" alt="LAB5_ss13" src="https://github.com/user-attachments/assets/eb05a3e0-82b5-4c76-b80b-c1fbe011920d" />


*Email modification request with administrator's CSRF token*

**Step 2: CSRF PoC Generation**
Used Burp's CSRF PoC generator to create an automated HTML form that:
- Pre-fills all required parameters (CSRF token, new email)
- Auto-submits on page load
- Can be hosted on the exploit server

**Step 3: Exploit Server Payload**
```html
<html>
  <body>
    <form action="https://[lab-id].web-security-academy.net/my-account/change-email" method="POST">
      <input type="hidden" name="csrf" value="[administrator-csrf-token]" />
      <input type="hidden" name="email" value="attacker@evil.com" />
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

**Step 4: Final Delivery**
Delivered the CSRF PoC to the victim via exploit server. When the administrator viewed the exploit:
1. Their browser auto-submitted the form
2. Request included valid administrative CSRF token
3. Server processed the email change as legitimate administrative action
4. Administrator's email changed to attacker-controlled address

### 14. Lab Completion - Account Takeover Achieved

The email modification succeeded, changing the administrator's email to the attacker-controlled address. This completed the full attack chain: **Cache Deception → Session Hijacking → CSRF Token Extraction → Account Takeover**.
<img width="1920" height="982" alt="LAB5_ss14" src="https://github.com/user-attachments/assets/4cbead18-c890-4e13-bc85-bb85d000aea8" />

*Administrator email successfully modified - Lab 5 solved*

**Attack Success Confirmation:**
- ✅ Cache successfully poisoned with administrative session
- ✅ CSRF token extracted from cached response
- ✅ Valid CSRF token used to bypass protection
- ✅ Administrator email modified without authentication
- ✅ Full account takeover possible via password reset

## Technical Impact

**Severity: Critical (CVSS 9.6)**

**Primary Vulnerabilities:**
1. **Web Cache Deception:** Exact match cache rules with delimiter normalization gaps
2. **Session Hijacking:** Cached authenticated content accessible to unauthorized users
3. **CSRF Token Leakage:** Anti-CSRF tokens exposed through cache poisoning
4. **Account Takeover:** Complete compromise of administrative accounts

**Attack Chain Impact:**
- **Authentication Bypass:** Complete circumvention of authentication through cached sessions
- **Privilege Escalation:** Regular user gains administrative CSRF tokens
- **Credential Takeover:** Email modification enables password reset and full account control
- **Persistence:** Attack repeatable during entire cache TTL window
- **Stealth:** Appears as legitimate static resource access in logs
- **Scalability:** Single cache poisoning affects all subsequent attackers

**Real-World Impact:** 
Organizations using CDNs, reverse proxies, or caching layers with exact match rules but inconsistent delimiter handling face critical risk. This vulnerability is particularly severe because:
- Administrative accounts are primary targets
- CSRF tokens are typically considered protected
- Cache poisoning enables multi-stage attacks
- Email modification leads to complete account takeover
- Detection is extremely difficult without cache-aware monitoring

## Remediation

### Immediate Fixes

1. **Eliminate delimiter normalization gaps:**
   - Apply identical delimiter parsing at cache and origin layers
   - Standardize semicolon (;) and question mark (?) handling
   - Reject ambiguous URLs with encoded path traversal patterns

2. **Remove exact match caching on dynamic paths:**
   - Never cache URLs matching dynamic endpoint patterns
   - Implement negative cache rules for `/my-account*` and similar sensitive paths
   - Use explicit allowlists for cacheable resources only

3. **Implement session-aware caching:**
   - Include authentication state in cache key generation
   - Set `Cache-Control: no-store, private` on all authenticated endpoints
   - Add `Vary: Cookie, Authorization` headers to prevent session leakage

4. **Enhanced CSRF protection:**
   - Implement double-submit cookie pattern
   - Add origin/referer validation
   - Use framework-native CSRF protection with proper configuration

### Long-Term Solutions

1. **Architecture redesign:**
   - Separate static and dynamic content delivery
   - Use different domains/subdomains for cached resources
   - Implement cache-aware application design patterns

2. **Comprehensive cache policy:**
   - Document all cacheable resources explicitly
   - Regular audits of cache configuration
   - Automated testing for cache deception vulnerabilities

3. **Security monitoring:**
   - Alert on cache hits for dynamic/authenticated endpoints
   - Monitor for unusual URL patterns with delimiters
   - Log cache key generation for security review
   - Detect rapid cache access patterns post-exploit-delivery

4. **Development practices:**
   - Include cache deception in threat modeling
   - Security training on cache/origin normalization risks
   - Penetration testing specifically for cache vulnerabilities
   - Code review for delimiter and normalization handling

## Key Takeaways

**Technical Skills Demonstrated:**
- Advanced fuzzing techniques for delimiter discovery
- Cache behavior analysis and exact match rule exploitation
- Multi-stage attack chain construction (deception → hijacking → CSRF → takeover)
- Session timing and synchronization exploitation
- CSRF token extraction and reuse
- Real-world attack automation and refinement

**Vulnerability Insights:**
- Exact match cache rules create false sense of security
- Delimiter handling inconsistencies are critical attack surfaces
- Commonly cached resources (robots.txt) become weaponizable
- Session-bound caching adds complexity but doesn't prevent exploitation
- Cache timing windows can be exploited with precise execution
- CSRF tokens must be protected from all exposure vectors, including cache

**Attack Complexity:**
This expert-level vulnerability required:
- Understanding of multiple security domains (caching, sessions, CSRF)
- Precise timing and synchronization
- Multi-step attack orchestration
- Creative combination of separate vulnerability classes
- Real-time adaptation to session constraints

## References

1. PortSwigger Web Security Academy - Web Cache Deception
2. RFC 7234 - HTTP/1.1 Caching
3. RFC 3986 - Uniform Resource Identifier (URI): Generic Syntax (Delimiter definitions)
4. Omer Gil - Web Cache Deception Attack (BlackHat 2017)
5. OWASP - Cross-Site Request Forgery (CSRF)
6. OWASP - Web Cache Deception Attack
7. RFC 9110 - HTTP Semantics (Cache-Control directives)

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** Techniques documented for authorized security testing and education only. Unauthorized system access is illegal. The author assumes no liability for misuse.
