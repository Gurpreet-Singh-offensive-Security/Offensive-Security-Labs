# Lab 3: Exploiting Origin Server Normalization for Web Cache Deception

## Executive Summary

This laboratory exercise demonstrates a critical web cache deception vulnerability resulting from inconsistent URL normalization between origin servers and caching layers. The exploitation leverages URL-encoded path traversal sequences combined with cacheable static resource prefixes to cause a web cache to store authenticated, user-specific content under a publicly accessible static resource path. This discrepancy enables unauthorized access to sensitive user data, including API keys, by unauthenticated attackers.

## Laboratory Metadata

| Attribute              | Value |
|------------------------|-------|
| **Source**             | PortSwigger Web Security Academy |
| **Difficulty Level**   | Practitioner (Medium) |
| **Vulnerability Class**| Web Cache Deception |
| **Risk Rating**        | High |
| **Lab Identifier**     | 0ab300ce03f31d8c80a6ee8800d100e3 |
| **Completion Date**    | December 23, 2025 |
| **Status**             | Solved |

## Technical Objective

The primary objective is to exploit normalization inconsistencies between the cache layer and origin server to deceive the caching mechanism into storing dynamic, authenticated content as if it were static. Successful exploitation demonstrates the ability to capture and retrieve sensitive user information (API keys) from the cached response without requiring authentication credentials.

## Testing Infrastructure

### Tools and Technologies
- **Burp Suite Professional** (Licensed to Gurpreet Singh)
  - HTTP Proxy for traffic interception
  - Intruder module for payload fuzzing
  - Repeater for request manipulation
  - HTTP History for traffic analysis
- **Browser Developer Tools** for client-side inspection
- **Exploit Server** for payload delivery mechanism

### Environment Configuration
- Laboratory environment accessed through Burp Suite configured as an intercepting proxy
- Authenticated test session established to validate sensitive data exposure at '/my-account' endpoint

## Vulnerability Analysis

### Root Cause
The vulnerability stems from a fundamental discrepancy in URL processing logic:
- **Cache Layer**: Keys responses based on raw, non-normalized URL paths
- **Origin Server**: Applies URL decoding and path normalization before routing requests

This inconsistency creates an exploitable gap where the cache perceives a request as targeting a static resource while the origin server resolves it to a dynamic, authenticated endpoint.

### Attack Surface
The '/my-account' endpoint exposes sensitive user information including usernames and API keys. This endpoint lacks appropriate cache control directives and is susceptible to cache poisoning through crafted URL paths.

## Exploitation Methodology

### Phase 1: Reconnaissance and Baseline Establishment

**Step 1: Traffic Interception**  
Configured Burp Suite proxy and accessed the laboratory environment to establish baseline traffic patterns.
<img width="1920" height="982" alt="LAB3_ss1" src="https://github.com/user-attachments/assets/5d491d99-f338-4dfc-9455-05f1d369f8fb" />

**Step 2: Sensitive Data Enumeration**  
Authenticated to the application and intercepted the '/my-account' response, confirming the exposure of username and API key in the HTML response body.
<img width="1920" height="982" alt="LAB3_ss2" src="https://github.com/user-attachments/assets/5a58e872-a781-4d1f-b9f3-b465eeedc908" />

**Step 3: Initial Path Behavior Testing**  
Issued a request to '/my-account/abc' to test arbitrary path handling. Result: HTTP 404 Not Found with no observable caching behavior, indicating strict path matching at the application layer.
<img width="1920" height="982" alt="LAB3_ss3" src="https://github.com/user-attachments/assets/a4e10bd7-8257-4874-9421-6f4f902aac25" />

### Phase 2: Delimiter and Normalization Testing

**Step 4: Delimiter Character Fuzzing**  
Deployed Burp Intruder in Sniper attack mode with a comprehensive delimiter payload set to identify characters that trigger alternate routing behavior. The query string delimiter ('?') returned HTTP 200 with API key visible, indicating potential for path manipulation.

<img width="1920" height="982" alt="LAB3_ss4" src="https://github.com/user-attachments/assets/e389c10c-8337-4322-8a7e-44e9388fa29b" />

**Step 5: URL Encoding Normalization Analysis**  
Submitted request to '/aaa/..%2Fmy-account' where '%2F' represents URL-encoded forward slash. The origin server decoded the sequence and performed path normalization, resulting in resolution to '/my-account' and serving the authenticated page. This confirms the server's normalization behavior while the cache treats the encoded sequence literally.
<img width="1920" height="982" alt="LAB3_ss5" src="https://github.com/user-attachments/assets/833be01a-8c91-45ca-8970-7329f5f72f4e" />


### Phase 3: Cache Behavior Characterization

**Step 6: Static Resource Path Identification**  
Analyzed HTTP history to identify paths with caching directives. The '/resources/ prefix exhibited Cache-Control headers with max-age values, indicating aggressive caching policy for static assets.
<img width="1920" height="982" alt="LAB3_ss6" src="https://github.com/user-attachments/assets/1a1138d9-2f5c-4672-bc8b-00259eb7a6b5" />

**Step 7: Cache Consistency Validation**  
Utilized Burp Repeater to send multiple requests to '/resources/' endpoints. Observed the expected cache miss on initial request followed by cache hits on subsequent requests, confirming predictable caching behavior.
<img width="1920" height="982" alt="LAB3_ss7 1" src="https://github.com/user-attachments/assets/fbb3d8f9-4888-44e5-80d9-e3343a6f1e7b" />
<img width="1920" height="982" alt="LAB3_ss7 2" src="https://github.com/user-attachments/assets/743347df-c83a-4ecd-9b53-23d90af44376" />

**Step 8: Cross-Authentication Cache Testing**  
Verified that '/resources/' content caching behavior remained consistent across both authenticated and unauthenticated sessions, establishing that the cache layer does not differentiate based on authentication state for this path prefix.

<img width="1920" height="982" alt="LAB_ss8" src="https://github.com/user-attachments/assets/fdd130f5-f80d-4edb-9f2d-d6ddbbc11e22" />


### Phase 4: Exploit Construction and Validation

**Step 9: Combined Exploit Path Development**  
Constructed request to '/resources/..%2Fmy-account' combining:
- Cacheable prefix ('/resources/') to trigger caching mechanism
- Encoded path traversal ('..%2F') to exploit normalization discrepancy
- Target endpoint ('my-account') to retrieve sensitive data

When accessed without authentication, the origin server responded with HTTP 302 redirect, while the cache layer treated the path as a static resource candidate.

<img width="1920" height="982" alt="LAB3_ss9" src="https://github.com/user-attachments/assets/00f66e7e-95d3-4878-9309-a662bc67bd17" />


**Step 10: Payload Delivery**  

Formulated the final exploit URL:
"https://<lab-id>.web-security-academy.net/resources/..%2Fmy-account"

Deployed the malicious URL via the Exploit Server to simulate a phishing or social engineering vector.
<img width="1920" height="982" alt="LAB3_ss10" src="https://github.com/user-attachments/assets/e0b98aa6-a74e-4909-abfc-706f72a4ffd5" />

### Phase 5: Cache Poisoning and Data Exfiltration

**Step 11: Victim Cache Poisoning**  
The authenticated victim (source IP: '10.0.3.29') accessed the exploit URL. The origin server normalized the path to '/my-account', served the authenticated response containing the victim's API key, and the cache layer stored this response under the literal path '/resources/..%2Fmy-account'.
<img width="1920" height="982" alt="LAB3_ss11" src="https://github.com/user-attachments/assets/59840b11-7ab4-4ba0-8bde-89e6b3c5046d" />

**Step 12: Unauthorized Data Retrieval**  
Issued an unauthenticated request to the same URL. The cache layer served the previously stored response containing the victim's API key, bypassing authentication requirements entirely.
<img width="1920" height="982" alt="LAB3_ss12" src="https://github.com/user-attachments/assets/2b62c9db-71fe-43b3-90ac-f740f40cb06c" />

**Step 13: Laboratory Completion**  
Successfully submitted the exfiltrated API key, demonstrating complete exploitation of the web cache deception vulnerability.
<img width="1920" height="982" alt="LAB3_ss13" src="https://github.com/user-attachments/assets/1887af91-9531-4faa-9c8f-19714222b97c" />



## Technical Impact Assessment

### Severity Factors
- **Confidentiality Impact**: High - Direct exposure of API keys and user credentials
- **Authentication Bypass**: Cache-based authentication circumvention
- **Data Persistence**: Cached sensitive data remains accessible for the duration of cache TTL
- **Attack Complexity**: Low - Requires only URL manipulation and social engineering
- **Detection Difficulty**: Medium - Appears as legitimate static resource access in logs

### Real-World Implications
This vulnerability class poses significant risks in production environments utilizing:
- Content Delivery Networks (CDNs)
- Reverse proxy caches
- Application-level caching layers
- Environments with inconsistent URL processing implementations

## Defense Mechanisms and Mitigations

### Immediate Remediation
1. **Unified URL Canonicalization**: Implement consistent URL normalization across all layers (cache, proxy, application)
2. **Strict Cache Control Headers**: Apply 'Cache-Control: no-store, private' directives to all authenticated and user-specific endpoints
3. **Path Validation**: Enforce strict allowlist-based path validation with rejection of encoded traversal sequences

### Strategic Security Enhancements
1. **Cache Key Normalization**: Configure cache to normalize URLs before key generation
2. **Authentication-Aware Caching**: Implement cache policies that consider authentication state and session context
3. **Path Prefix Isolation**: Avoid blanket caching rules based solely on path prefixes or file extensions
4. **Security Testing**: Incorporate cache deception testing into security assessment protocols

## Knowledge Advancement

### Key Technical Insights
- URL normalization discrepancies between architectural layers create exploitable security gaps
- Cache keying mechanisms must account for semantic path equivalence, not just literal string matching
- Path prefixes designated as "static" can be weaponized to cache dynamic content
- The combination of encoded traversal with cacheable prefixes creates a powerful exploitation primitive

### Professional Development
This laboratory exercise significantly enhanced understanding of:
- Advanced cache deception techniques beyond basic exploitation
- Systematic methodology for testing normalization behavior
- Integration of fuzzing techniques with manual testing
- Real-world attack scenario simulation and payload delivery

**Complexity Rating**: Intermediate, requiring understanding of HTTP caching, URL processing, and web application architecture

## References and Further Reading

1. PortSwigger Web Security Academy. (2025). *Web Cache Deception*. Retrieved from https://portswigger.net/web-security/web-cache-deception
2. OWASP Foundation. (2025). *Web Cache Deception Attack*. Retrieved from https://owasp.org/www-community/attacks/Web_Cache_Deception_Attack
3. Gil, O. (2017). *Web Cache Deception Attack*. BlackHat USA 2017.
4. RFC 3986. *Uniform Resource Identifier (URI): Generic Syntax*.
5. RFC 7234. *Hypertext Transfer Protocol (HTTP/1.1): Caching*.

## Acknowledgments

Laboratory environment provided by PortSwigger Web Security Academy. Testing conducted in an authorized, controlled environment for educational purposes.

---

## Legal Notice

**Copyright © 2025 Gurpreet Singh**  
All rights reserved.

**License**: Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)

**Source Repository**: https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Academic Integrity Statement**: This document represents original research and analysis. Unauthorized reproduction, plagiarism, or submission of this content as one's own work constitutes academic dishonesty and potential copyright infringement. Proper attribution must be provided when referencing this material.

**Disclaimer**: The techniques described herein are intended solely for authorized security testing and educational purposes. Unauthorized access to computer systems is illegal. The author assumes no liability for misuse of this information.
