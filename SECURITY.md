# Security & Peer Review

This repository contains personal writeups and methodology for
PortSwigger Web Security Academy labs. Since it's documentation
rather than software, this policy focuses on two things: OPSEC
slip-ups and technical peer review.

---

## OPSEC & Data Leaks

Even with careful scrubbing, things get missed. If you spot any
of the following in screenshots or HTTP history captures, treat
it as a sensitive disclosure:

- A live session cookie, JWT, or CSRF token left unredacted
- An active API key or credential visible in Burp output
- A linked domain that has since been taken over

**To report:** Email gskhalsa6245@gmail.com with the subject
`[OPSEC] Lab <number> — <brief description>`. Point to the
file and screenshot. Do not open a public issue — I'll
rewrite the Git history to scrub it.

---

## Peer Review & Payload Optimization

If you're an AppSec engineer, pentester, or bug bounty hunter
and you read a writeup thinking:

- *"There's a cleaner payload for this."*
- *"This bypasses X but breaks against Y WAF."*
- *"You missed an interesting edge case here."*

I want to hear it.

**Open a GitHub Issue** to discuss an alternative approach, or
**submit a PR** to add an "Alternative Approach" section to a
writeup. I'll review it, merge it if it's solid, and credit you.

---

## What This Repo Is Not

The documented vulnerabilities are intentional educational
content — they are not security issues to report. If you're
looking at a writeup describing an injection technique, that's
the point of the writeup.

---

## Ethics & Legal

All techniques documented here are for authorized testing and
educational research only. Do not use these against systems
you don't own or lack explicit written permission to test.

**Maintainer:** Gurpreet Singh — Offensive Security Researcher
**Contact:** gskhalsa6245@gmail.com
