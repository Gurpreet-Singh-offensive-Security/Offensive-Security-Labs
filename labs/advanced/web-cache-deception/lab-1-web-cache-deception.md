# Lab 1: Web Cache Deception

## Lab Information
- **Lab Title**: Web Cache Deception
- **Source**: PortSwigger Web Security Academy
- **Difficulty**: Practitioner (Medium)
- **Category**: Advanced Techniques / Web Cache Poisoning
- **Impact**: High (Exposure of sensitive user data like API keys)
- **Lab ID**: 0a1e0073049b2b6680a8039500110012 (your specific instance)
- **Completion Date**: December 20, 2025
- **Tools Used**: Burp Suite Professional, Browser DevTools, Exploit Server
- **Status**: Solved

## Objective
The goal of this lab is to exploit a web cache deception vulnerability to access a victim's sensitive information (API key) by tricking the cache into storing a dynamic user page as a static resource. This is achieved by leveraging server path normalization discrepancies and cache behavior on seemingly static file extensions (e.g., .js or .css).

## Reconnaissance
- Verified Burp Suite Professional license under user "Gurpreet Singh" and prepared the web interface for testing.
- Logged in to a test account and identified sensitive information exposure: API keys visible in the account page (/my-account).
- Tested server path handling: Changing /account to /account/test redirected back to /account without errors, indicating potential path normalization issues.
- Analyzed caching behavior:
  - Requested /my-account/foo.css → Cache-Control: max-age=30, Age: 0 (cache miss).
  - Re-requested within 30 seconds → Cache hit.
- This confirmed the server/cache treats certain paths with static extensions as cacheable, even if they serve dynamic content due to path stripping.

## Exploitation Steps
1. **Prepare the Environment**: Confirmed Burp Pro setup and lab readiness.
   <img width="1920" height="982" alt="LAB1_ ss1" src="https://github.com/user-attachments/assets/cf6557fb-78f8-4666-ab5e-55fff6cb25ad" />

2. **Identify Sensitive Data**: Logged in and found API keys exposed in the account page.
  <img width="1920" height="982" alt="LAB1_ss2" src="https://github.com/user-attachments/assets/20ea3c5b-721e-4a78-a80b-10217fcf9ac5" />

   
3. **Test Path Handling**: Modified path from /account to /account/test; server redirected seamlessly, revealing normalization vulnerability.
   <img width="1920" height="982" alt="LAB1_ss3" src="https://github.com/user-attachments/assets/d09e0941-0d56-42e2-8914-c93942dc485d" />

   

4. **Verify Caching**: Tested /my-account/foo.css for cache misses/hits with max-age=30.
   <img width="1920" height="982" alt="LAB1_ss4" src="https://github.com/user-attachments/assets/6fcadca7-ee4c-4653-bbfe-f8b4fe137e85" />
   <img width="1920" height="982" alt="LAB1_ss5" src="https://github.com/user-attachments/assets/bb4dd094-bec5-4a4c-84b8-69b2ef3a94cf" />


5. **Craft Exploit Payload**: Created an exploit to force the victim to request a deceptive URL (/my-account/wcd.js), which the server normalizes to /my-account but caches as a static .js file.
   - Payload: ``<script>document.location="https://0a1e0073049b2b6680a8039500110012.web-security-academy.net/my-account/wcd.js"</script>`
   - Delivered via Exploit Server.
    <img width="1920" height="982" alt="LAB1_ss6" src="https://github.com/user-attachments/assets/43f8bc35-bd91-4359-901f-375d3fd3314a" />

     
6. **Deliver Exploit to Victim**: Victim clicks the exploit link, executes the script, and requests the deceptive URL, caching their account page (with API key).

     <img width="1920" height="982" alt="LAB1_ss7" src="https://github.com/user-attachments/assets/6fe03bda-7fad-4370-94d9-3c4a5a2856c4" />

   
7. **Retrieve Cached Data**: Accessed the cached /my-account/wcd.js as attacker, obtaining the victim's API key.
   
      <img width="1920" height="982" alt="LAB1_ss8 1" src="https://github.com/user-attachments/assets/475ba6d3-28d4-41be-a1c2-6220297fea4e" />


8. **Submit Solution**: Submitted the API key to complete the lab.

      <img width="1920" height="982" alt="LAB1_ss8 2" src="https://github.com/user-attachments/assets/d28abe18-4e9f-4cca-881c-211db63f1d09" />


   

## Key Concepts
- **Web Cache Deception**: Occurs when a web cache is tricked into storing sensitive dynamic pages as static resources due to URL parsing differences between the cache and backend server.
- **Path Normalization**: Servers may strip or ignore file extensions/paths, serving dynamic content but allowing caching if the URL looks static.
- **Cache Keying**: Caches often key on the full URL, including ignored parts, enabling attackers to store/retrieve user-specific data.
- **Exploit Delivery**: Use XSS-like payloads or redirects to force authenticated users to cache their data.

## Mitigation Strategies
- **Normalize URLs Properly**: Ensure backend servers reject or 404 invalid paths instead of redirecting/stripping.
- **Cache-Control Headers**: Use `no-store` or `private` on sensitive pages; avoid caching dynamic content.
- **Extension Whitelisting**: Only cache specific static extensions (e.g., .css, .js) from known static directories.
- **Path Validation**: Implement strict URL parsing to prevent deception.
- **Monitoring**: Log and alert on suspicious cache requests.

## References
- PortSwigger Web Security Academy: [Web Cache Deception](https://portswigger.net/web-security/web-cache-poisoning/web-cache-deception)
- OWASP: [Web Cache Deception Attack](https://owasp.org/www-community/attacks/Web_Cache_Deception_Attack)
- Research Paper: "Web Cache Deception Escalates" by Omer Gil

## Personal Notes
This was my first lab, and it highlighted the importance of understanding cache behaviors in web apps. The path discrepancy was subtle but key to the exploit. Next time, I'll test more extensions (.css, .jpg) for variations. Time taken: ~2 hours. Great intro to advanced client-side attacks—excited for more!
