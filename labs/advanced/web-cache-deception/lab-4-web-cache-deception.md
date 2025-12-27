# Lab 4: Exploiting Cache Server Normalization for Web Cache Deception

## Executive Summary

This lab demonstrates a web cache deception vulnerability caused by normalization inconsistencies between the cache layer and origin server. By exploiting delimiter handling differences and URL-encoded path traversal, I successfully forced the cache to store authenticated user data under a publicly accessible cache key, enabling API key extraction without authentication.

## Lab Details

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty**         | Practitioner |
| **Vulnerability**      | Web Cache Deception via Server Normalization |
| **Risk**               | High - Authentication Bypass & Credential Exposure |
| **Completion**         | December 24, 2025 |

## Objective

Exploit cache server normalization discrepancies to poison the cache with authenticated content, then retrieve a victim's API key without authentication.

## Testing Setup

**Tools Used:**
- Burp Suite Professional (Licensed to Gurpreet Singh)
- Exploit Server for payload delivery
- Browser for session management

**Target:** "/my-account" endpoint containing user API keys

## Exploitation Walkthrough

### 1. Initial Reconnaissance

Started by intercepting traffic through Burp Suite and identifying the target endpoint. Authenticated to the application and accessed "/my-account" to confirm sensitive data exposure.

<img width="1920" height="983" alt="LAB4_ss1" src="https://github.com/user-attachments/assets/86f79a73-7412-4fdb-8b46-488256cfa035" />

*Lab environment with Burp Suite Professional configured*
<img width="1920" height="983" alt="LAB4_ss2" src="https://github.com/user-attachments/assets/0f61e5dc-1b09-4589-9d66-c8c16b15ab3f" />

*API key visible in "/my-account" endpoint response*

### 2. Path Behavior Testing

Tested basic path manipulation by appending "/abc" to "/my-account". Received a 404 response with no caching headers, indicating strict path matching without default cache behavior
<img width="1920" height="983" alt="LAB4_ss3" src="https://github.com/user-attachments/assets/2cee2c67-0a9c-4978-857b-5a72c6aa51dc" />

*Strict path matching confirmed - no normalization or caching on arbitrary paths*

### 3. Delimiter Discovery

Used Burp Intruder with a delimiter payload list to identify characters the server recognizes as path delimiters. **Critical step:** Disabled URL encoding in payload settings to test both raw and encoded delimiters.

**Discovered delimiters returning 302 redirects:**
- Raw: '#', '/', '?'
- Encoded: '%23', '%2F', '%3F'

<img width="1920" height="983" alt="LAB4_ss4" src="https://github.com/user-attachments/assets/1e2d702b-666c-44bc-841f-f2138052f0f9" />

*Intruder results showing delimiter candidates (302 responses highlighted)*

### 4. Delimiter Validation

Tested each delimiter individually in Repeater. Example: "/my-account%3Fabc" returned 302 redirects but no cache headers, confirming the server decodes and recognizes delimiters but doesn't automatically cache them.
<img width="1920" height="983" alt="LAB4_ss5" src="https://github.com/user-attachments/assets/6c105c4f-01db-4789-bfa0-0a967979107d" />

*URL-encoded delimiter testing - server recognizes but doesn't cache*

### 5. Static Resource Identification

Analyzed HTTP history and identified that all requests to "/resources/" prefix consistently returned cache headers ('Cache-Control', 'X-Cache'), indicating aggressive caching policies.

<img width="1920" height="983" alt="LAB4_ss6 1" src="https://github.com/user-attachments/assets/e1276b6f-ad13-4949-a100-bb8db342b7c0" />

*HTTP history showing cache headers on '/resources/' paths*

### 6. Cache Behavior Validation

Tested cache behavior with path traversal: '/aaa/..%2Fresources'

**First request:** 404 with 'X-Cache: miss' - cache attempted to store based on literal path containing "resources"
<img width="1920" height="983" alt="LAB4_ss6 2" src="https://github.com/user-attachments/assets/40d6588e-9e67-463e-8ab2-95e6d9010bdc" />

*Cache miss on first request*

**Second request:** Same path returned 'X-Cache: hit', confirming the cache uses raw URLs as keys and applies policies based on prefix matching.
<img width="1920" height="983" alt="LAB4_ss6 3" src="https://github.com/user-attachments/assets/ebca8461-1b6e-41fb-8fb2-4a978cb4f533" />

*Cache hit on subsequent request*

### 7. Exploit Construction

Combined findings to craft the attack URL:

/my-account%23%2F%2E%2E%2Fresources


**How it works:**
- **Cache sees:** Literal path containing "resources" → applies caching
- **Origin sees:** Decodes to "/my-account#/../resources → normalizes to /my-account"

Tested in Repeater and received 302 with 'X-Cache: hit', confirming successful cache poisoning setup.

<img width="1920" height="983" alt="LAb4_ss7" src="https://github.com/user-attachments/assets/cc5c2c0e-f6c3-494b-8dbe-91a5cfa39a5b" />

*Exploit path returning 302 with cache hit*

Configured exploit server with malicious URL for victim delivery.

*Exploit server ready for delivery*

### 8. Cache Poisoning

Delivered exploit to victim. Access logs confirmed the authenticated victim (IP: '10.0.4.188') clicked the link, causing the cache to store their authenticated '/my-account' response under the crafted cache key.
<img width="1920" height="983" alt="LAB4_ss8" src="https://github.com/user-attachments/assets/dee087b2-d9fb-475c-b455-ad6a19ed7b51" />

*Victim accessed exploit URL - cache poisoned with authenticated content*

### 9. API Key Extraction

Accessed the same URL without authentication. The cache served the victim's authenticated response, exposing their API key. Submitted the key to complete the lab.
<img width="1920" height="983" alt="LAB4_ss9 1" src="https://github.com/user-attachments/assets/931e71ea-d368-4015-92de-49bb75a6dfd7" />



*Retrieved victim's API key from cache - lab solved*

## Technical Impact

**Severity: High (CVSS 8.1)**

- **Authentication Bypass:** Complete circumvention of access controls via cache layer
- **Credential Exposure:** Direct extraction of API keys and sensitive user data
- **Persistence:** Cached data remains accessible for entire TTL (up to 1 year in this case)
- **Scalability:** Single poisoned entry accessible to multiple attackers
- **Detection Difficulty:** Appears as legitimate static resource access in logs

**Real-World Impact:** Organizations using CDNs, reverse proxies, or caching layers with inconsistent URL normalization face this risk. Compromised API keys enable full account takeover and unauthorized system access.

## Remediation

### Immediate Fixes
1. **Unified normalization:** Apply identical URL decoding/normalization at cache and origin layers
2. **Strict cache headers:** Set 'Cache-Control: no-store, private' on all authenticated endpoints
3. **Path validation:** Reject encoded traversal sequences and suspicious delimiter combinations

### Long-Term Solutions
1. **Authentication-aware caching:** Include auth state in cache keys
2. **Explicit allowlists:** Cache only explicitly defined static resources, not path prefixes
3. **Monitoring:** Alert on cache hits for dynamic endpoints or unusual encoded patterns
4. **Security testing:** Include cache deception in regular pentesting scope

## Key Takeaways

**Technical Skills Demonstrated:**
- Systematic fuzzing to identify behavioral discrepancies
- Cache behavior analysis and exploitation
- Multi-layer attack chain construction
- Real-world attack simulation

**Vulnerability Insights:**
- Cache/origin normalization gaps create serious security holes
- Static path prefixes can be weaponized for cache poisoning
- Delimiter handling inconsistencies enable sophisticated bypasses
- Proper cache key generation requires semantic URL understanding

## References

1. PortSwigger Web Security Academy - Web Cache Deception
2. RFC 7234 - HTTP/1.1 Caching
3. Omer Gil - Web Cache Deception Attack (BlackHat 2017)
4. OWASP - Web Cache Deception Attack

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0

**Repository:** https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Disclaimer:** Techniques documented for authorized security testing and education only. Unauthorized system access is illegal. The author assumes no liability for misuse.
