# 🕸️ Web Cache Deception — Complete Lab Series

<div align="center">

![Category](https://img.shields.io/badge/Category-Web%20Cache%20Deception-8E44AD?style=for-the-badge)
![Labs Completed](https://img.shields.io/badge/Labs%20Completed-5%2F5-00C853?style=for-the-badge)
![Difficulty](https://img.shields.io/badge/Difficulty-Apprentice%20to%20Expert-FF6D00?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-100%25%20Complete-00C853?style=for-the-badge)

**5 labs. Apprentice through Expert. All done.**

[![🔗 Main Repository](https://img.shields.io/badge/🔗%20Main%20Repository-black?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs)  [![👤 About Me](https://img.shields.io/badge/👤%20About%20Me-5865F2?style=for-the-badge)](https://github.com/Gurpreet-Singh-offensive-Security)  [![📧 Contact](https://img.shields.io/badge/📧%20Contact-E8562A?style=for-the-badge)](mailto:gskhalsa6245@gmail.com)

</div>

---

Finished all 5 PortSwigger Web Cache Deception labs. Notes below cover what I actually did — not rephrased documentation. The attack patterns section is what I kept open while working through Labs 3–5.

**Tools:** Burp Suite Pro, exploit server  
**Core concept:** Cache and origin disagree on what a URL points to. Attacker exploits that gap.

---

## What's Actually Happening

The cache and the origin server parse URLs differently. If you can craft a URL that the cache treats as a static resource but the origin serves as authenticated dynamic content — the cache stores the authenticated response. Anyone who requests that URL after gets the cached sensitive data without needing to be logged in.

The five variations in this series are just different ways to create that parsing gap: extensions, delimiters, encoding, path traversal, and exact-match rule abuse.

---

## Labs

### Apprentice — 1/1

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 1 | [Static Extension Cache Poisoning](./lab-1-web-cache-deception.md) | Appended `.js` to the `/my-account` path. Cache saw a static JS file and cached the response. Origin ignored the extension and served the authenticated account page. Sent the crafted URL to the victim via exploit server. When they clicked it, the response got cached. Fetched the same URL without auth — got their API key. | Path Normalization |

---

### Practitioner — 3/3

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 2 | [Delimiter-Based Cache Deception](./lab-2-web-cache-deception.md) | Fuzzed delimiters to find ones the cache and origin handle differently. Found that `;` worked — cache treated `/my-account;static.js` as a static resource, origin stripped everything after `;` and served `/my-account`. Sent URL to victim, retrieved cached response. | Delimiter Discrepancy |
| 3 | [URL Encoding + Path Traversal](./lab-3-web-cache-deception.md) | Delimiter fuzzing alone didn't work here. Combined URL encoding with path traversal — used `%2f` to slip past the cache's path matching. Found the cache key and variable via HTTP history. Built a combined payload that the cache keyed as static but the origin resolved to the account endpoint. | Encoding + Traversal |
| 4 | [Delimiter Discrepancy at Static Endpoint](./lab-4-web-cache-deception.md) | Cache had a rule matching a specific static path. Added a delimiter right at that endpoint so the cache matched its own rule, then appended a traversal that the origin resolved to the target. Found the cache path by checking what was already being cached in HTTP history. Confirmed the discrepancy and built the exploit. | Delimiter + Cache Path Abuse |

---

### Expert — 1/1

| # | Lab | What I did | Technique |
|---|-----|-----------|-----------|
| 5 | [Exact Match Rules + CSRF Account Takeover](./lab-5-web-cache-deception.md) | Exact match cache rules — most techniques failed. Found a cacheable path via `robots.txt`. Combined URL encoding with the cache path to get the authenticated response cached. Timing was the tricky part — used a cache buster to reset state when needed. Once I had the victim's CSRF token from the cached response, generated a CSRF PoC in Burp, sent the exploit link to the victim, and used the captured token to make account changes on their behalf. Full account takeover. | Encoding + CSRF Chain |

---

## How Each Attack Works

### Extension / Path Normalization

Simplest variant. Cache rules match on file extension. Appending `.js` or `.css` to a dynamic endpoint makes the cache think it's a static asset.

```
/my-account.js

Cache sees:  .js extension → cache this
Origin sees: /my-account   → serve authenticated page
Result:      Authenticated page stored as public cache entry
```

---

### Delimiter Discrepancy

Cache and origin handle delimiter characters differently. The goal is finding a character that truncates the path for the origin but doesn't for the cache (or vice versa).

```
/my-account;foo.js

Cache sees:  /my-account;foo.js → matches static rule → cache
Origin sees: /my-account        → ';' is a delimiter → serve auth page
```

Fuzzing workflow: Burp Intruder with a delimiter wordlist (`?`, `;`, `#`, `!`, `~`, `@`, `$`...) against the target endpoint. Look for responses that differ — same content as the clean endpoint but with cache headers.

---

### URL Encoding + Path Traversal

When raw delimiters are blocked or normalised consistently, try encoded variants. Cache often doesn't decode before matching rules.

```
/static/..%2fmy-account

Cache sees:  /static/..%2fmy-account → matches /static/ prefix → cache
Origin sees: /my-account             → decoded and traversed → serve auth page
```

Check HTTP history for what the app is already caching — gives you the cache key format to target.

---

### Exact Match Rule Abuse

Harder to exploit because the cache only caches specific exact paths. `robots.txt` or known static files are the starting point — if you can find a cacheable path and combine it with encoding tricks that the origin resolves differently, you still get the gap.

```
robots.txt → cacheable
/robots.txt%2f..%2fmy-account → cache keys on decoded path matching robots.txt rule
                               → origin resolves full traversal → auth content cached
```

Cache buster tip: add a unique query param (`?cb=1234`) when testing to avoid hitting a previous cached response and getting a false result.

---

### CSRF Token Extraction via Cached Response

Once you have a cached authenticated response containing a CSRF token — you can use it. The flow for Lab 5:

1. Craft URL that causes victim's authenticated response to get cached
2. Send to victim (exploit server link)
3. Victim visits → their response cached
4. Fetch the cached URL without auth → extract CSRF token from response
5. Burp → Generate CSRF PoC with the stolen token
6. Send PoC to victim → action executes in their session

---

## Delimiter Fuzzing Reference

Characters worth testing (raw and URL-encoded):

```
Raw:     ? ; # ! ~ @ $ % & + = | \ ^ ` ' "
Encoded: %3f %3b %23 %21 %7e %40 %24 %25 %26 %2b %3d %7c %5c %5e %60 %27 %22
Double:  %253f %253b %2523 (double-encode for WAF bypass or strict decoders)
```

Set up Intruder in Sniper mode, mark the delimiter position, load the list. Filter on responses that return 200 with the same body as the clean endpoint but include cache hit headers (`X-Cache: hit`, `Age:`, `CF-Cache-Status: HIT`).

---

## Fix

The root cause in every lab: the cache and origin server don't agree on what a URL means.

**Cache layer:** Don't use prefix matching for cache rules — it's too easy to exploit with path traversal. Use explicit exact-match allowlists for what gets cached. If a response contains auth-related data, it shouldn't be cached regardless of the URL.

**Origin server:** Set `Cache-Control: no-store, private` on every authenticated endpoint. Don't rely on the cache layer to figure out what's sensitive.

**Both layers:** Normalise URLs the same way before any routing or caching decision. If the cache decodes `%2f` to `/` before matching, the origin needs to do the same — and vice versa.

**Monitoring:** Cache hits on endpoints that should never be cached (account pages, API responses with tokens) should alert. Unusual URL patterns with encoded characters or delimiter sequences in paths are worth logging.

---

## Tools

| Tool | What I used it for |
|------|--------------------|
| Burp Suite Pro | Repeater for manual testing, Intruder for delimiter fuzzing |
| Burp CSRF PoC Generator | Lab 5 — generating the account takeover payload |
| Exploit server | Delivering links to victim, hosting CSRF PoC |
| HTTP History | Finding existing cached paths and cache key format |

---

## Lab Order

Do them in order. Labs 1–2 build the core mental model — without understanding the cache/origin gap clearly, Labs 3–5 won't make sense. Lab 3 and 4 are where you start combining techniques. Lab 5 is the hardest, mostly because of the timing and the CSRF chain at the end.

---

## Completion

- [x] Lab 1 — Static Extension Cache Poisoning
- [x] Lab 2 — Delimiter-Based Cache Deception
- [x] Lab 3 — URL Encoding + Path Traversal
- [x] Lab 4 — Delimiter Discrepancy at Static Endpoint
- [x] Lab 5 — Exact Match Rules + CSRF Account Takeover

---

## Resources

- [PortSwigger Web Cache Deception](https://portswigger.net/web-security/web-cache-deception)
- [Omer Gil — Web Cache Deception Attack (Black Hat 2017)](https://omergil.blogspot.com/2017/02/web-cache-deception-attack.html) — the original research
- [James Kettle — Practical Web Cache Poisoning (Black Hat 2018)](https://portswigger.net/research/practical-web-cache-poisoning) — closely related, worth reading together

Other series in this repo:
- [SQL Injection](../SQL-Injection/) — 18 labs done
- [Authentication Vulnerabilities](../Authentication/) — 14 labs done
- XSS — in progress
- Access Control — in progress

---

## Legal

Done entirely on PortSwigger Academy's lab environment. Using any of this against systems you don't own or don't have written permission to test is illegal.

---

<div align="center">

### Gurpreet Singh | Offensive Security Researcher

[![EMAIL](https://img.shields.io/badge/EMAIL-GSKHALSA6245%40GMAIL.COM-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:gskhalsa6245@gmail.com)

[![GITHUB](https://img.shields.io/badge/GITHUB-GURPREET--SINGH--OFFENSIVE--SECURITY-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/Gurpreet-Singh-offensive-Security/Offensive-Security-Labs) [![LINKEDIN](https://img.shields.io/badge/LINKEDIN-CONNECT-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/gurpreetsingh-security)

**CC BY-NC-ND 4.0 — © 2025 Gurpreet Singh**

</div>
