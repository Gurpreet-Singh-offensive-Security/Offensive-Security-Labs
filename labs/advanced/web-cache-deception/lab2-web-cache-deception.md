Lab 2: Exploiting Path Delimiters for Web Cache Deception

Lab Information

Lab Title: Web Cache Deception
Source: PortSwigger Web Security Academy
Difficulty: Practitioner(Medium)
Category: Advanced Techniques/ Web Cache Deception
Impact: High(EExposure of sensitive user data like API keys)
Lab ID: 0a1e0074049b2b6680a8039500110012
Completion Date: December 20, 2025
Tools Used: Burp Suite Professional, Browser Dev Tools, Exploit Server
Status: Solved


Objective

Exploit delimiter discrepancies between cache and origin server to perform web cache deception attack and extract victim's API key.

---

Vulnerability Overview

Path delimiters are characters that separate or terminate URL path segments. This vulnerability occurs when the cache server and origin server interpret these delimiters differently, allowing an attacker to cache dynamic content as if it were static.

Attack Principle:

URL: /my-account;.css

Origin Server → Stops at ' ; ' → Returns /my-account content
Cache Server  → Ignores ' ; ' → Caches as static .css file

---

Lab Setup

<img width="1920" height="982" alt="LAB2_ss1" src="https://github.com/user-attachments/assets/872a9f4f-b720-422c-8657-e09af2ef64b6" />


Tools Used:

- Burp Suite Professional
- Proxy configuration: 127.0.0.1:8080
- Target: PortSwigger Web Security Academy

---

Exploitation Process

1. Reconnaissance

<img width="1920" height="982" alt="LAB2_ss2" src="https://github.com/user-attachments/assets/85c0cfed-eb5f-4cff-bc3b-0128c51c4543" />


Identified that " /my-account" endpoint exposes sensitive API key in the response. This authenticated endpoint becomes the target for cache deception.

2. Control Testing

<img width="1920" height="982" alt="LAB2_ss3" src="https://github.com/user-attachments/assets/b6a7138c-5c10-4f78-8b20-6896555fd03e" />


Tested baseline behavior with invalid path " /my-account ":
- Result: 400 Bad Request
- No cache interaction
- Confirms strict path validation

This establishes that simple path manipulation fails, requiring delimiter-based exploitation.

3. Delimiter Discovery

<img width="1920" height="982" alt="LAB2_ss4" src="https://github.com/user-attachments/assets/d8634b70-5782-42a2-bb26-f05f2b70827e" />

Configuration:

- Attack type: Sniper
- Payload position: "/my-account§§abc"
- URL encoding: Disabled
- Payloads: ASCII delimiter characters ( ; ,  ? ,  / , # , % , \ , etc.)

Rationale: Systematic testing of common delimiters used across different frameworks (Java, .NET, PHP) and CDN implementations.

4. Results Analysis

<img width="1920" height="982" alt="LAB2_ss5" src="https://github.com/user-attachments/assets/4a68edca-692f-4808-ba04-adeb01c333d9" />


Findings:

| Delimiter | Status | Behavior |
|-----------|--------|----------|
|    ;      | 302 | Server stops parsing at delimiter |
|    ?      | 302 | Server stops parsing at delimiter |
|    /      | 302 | Server stops parsing at delimiter |
|   Others  | 404 | Rejected |

Three delimiters successfully create parsing discrepancies between cache and server.

### 5. Cache Behavior Analysis
<img width="1920" height="982" alt="LAB2_ss6 1" src="https://github.com/user-attachments/assets/206a409d-8820-4581-a895-3bb4aa2d3865" />

<img width="1920" height="982" alt="LAB2_ss6 2" src="https://github.com/user-attachments/assets/bbcfb802-2db7-4ead-8947-63ce8b33559c" />


Cache Response Testing:

Semicolon (;):

Request: /my-account;.css  
X-Cache: miss → hit (cached)  
Result:  Exploitable

Question Mark (?):

Request: /my-account?.css
X-Cache: No caching
Result:  Not exploitable (recognized as query string)

Forward Slash ( / ):

Request: /my-account/.css
X-Cache: miss → hit (cached)
Result:  Exploitable


Selected delimiter: ; (most reliable across frameworks)

6. Exploit Delivery

<img width="1920" height="982" alt="LAB2_ss7" src="https://github.com/user-attachments/assets/cab3419b-35fe-4049-9924-5ecc942f6b54" />

Attack Flow:

1. Victim clicks exploit link (while authenticated)
2. Request sent: /my-account;.css with victim's session
3. Server returns dynamic content (API key included)
4. Cache stores response as static .css file
5. Attacker requests same URL without authentication
6. Cache serves victim's data

Real-world delivery methods: Phishing emails, compromised websites, malvertising, social engineering

7. Exploitation Validation

<img width="1920" height="982" alt="LAB2_ss8" src="https://github.com/user-attachments/assets/b1c2ae72-6575-430f-9f70-f4c13d75e90c" />

Access logs confirm:
- Victim accessed the malicious URL
- Authenticated session present
- Response cached (X-Cache: miss)
- Cache poisoning successful

8. Data Extraction

<img width="1920" height="982" alt="LAB2_ss9" src="https://github.com/user-attachments/assets/655f8a09-0803-4832-9605-ec780e7da6a6" />

Successfully retrieved victim's API key from cached response without authentication.

Security Impact:
- Unauthorized access to sensitive credentials
- Bypassed authentication controls
- API key can be used for privileged operations

9. Lab Completion
<img width="1920" height="982" alt="LAB2_ss9 2" src="https://github.com/user-attachments/assets/6ef3f05c-60c9-4b3a-ac2a-6dd86d83fc4a" />

Lab successfully completed by submitting extracted API key.

---

Technical Analysis

Why This Works

Server Behavior
- Interprets " ; " as delimiter (common in Java/Spring, .NET)
- Processes only /my-account portion
- Returns authenticated user content with 302 redirect

Cache Behavior:
- Ignores " ; " character
- Sees complete path /my-account;.css
- Treats as static CSS file based on extension
- Caches response with Cache-Control: max-age=30

Delimiter Research

Common delimiters across frameworks:

| Framework   | Delimiters | Example |
|-------------|------------|---------|
| Java/Spring | ;          | /path;param=value |
| .NET/IIS    | ;          | /path;.aspx |
| PHP         | ; (varies) | /path;.php |

Testing methodology included both raw and encoded variants (%3b, %2f) to identify parsing inconsistencies.

---

Detection & Prevention

Detection
- Monitor for delimiter characters followed by static extensions on dynamic paths
- Alert on cache HITs for authenticated endpoints
- Analyze X-Cache headers for unexpected patterns


Prevention
1. Normalize paths consistently between cache and origin
2. Implement strict cache rules - whitelist cacheable paths
3. Use cache deception armor (CDN-level protection)
4. Never cache authenticated content.


Security Headers:


Cache-Control: no-store, private
X-Content-Type-Options: nosniff

Key Takeaways

- Delimiter interpretation varies between cache layers and application servers
- Systematic testing reveals exploitable parsing discrepancies
- Framework knowledge helps identify likely delimiters
- Cache behavior analysis is critical for successful exploitation
- Real-world attacks combine technical exploitation with social engineering

---

## References

- PortSwigger Web Security Academy: [Web Cache Deception](https://portswigger.net/web-security/web-cache-poisoning/web-cache-deception)
- OWASP: [Web Cache Deception Attack](https://owasp.org/www-community/attacks/Web_Cache_Deception_Attack)
- Omer Gil - "Web Cache Deception Attack" (2017)
- PortSwigger Research - "Gotta Cache 'em All" (Black Hat USA 2024)

---
## Copyright & License

**© 2025 Gurpreet Singh**  
Licensed under CC BY-NC-ND 4.0  
Original source: https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs

**Plagiarism Notice:** This content is protected. Unauthorized copying or submission as your own work is prohibited and constitutes academic dishonesty.

