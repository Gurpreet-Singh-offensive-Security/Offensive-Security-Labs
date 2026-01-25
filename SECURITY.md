# Security Policy

## Overview

Offensive Security Labs is an educational project where I document web application vulnerabilities and penetration testing techniques. This policy outlines how to responsibly report security issues related to this repository.

## Scope

This policy applies to:

- ✅ Documentation and lab writeups in this repository
- ✅ Python automation scripts and tools
- ✅ Repository configuration and workflows
- ✅ Any code or content I maintain within this project

This policy does **NOT** apply to:

- ❌ Third-party platforms referenced in labs (e.g., PortSwigger Academy)
- ❌ Vulnerabilities intentionally documented as educational content
- ❌ External tools or libraries mentioned in the documentation

## Supported Versions

As this is an educational repository with continuous updates, only the latest version on the `main` branch is actively maintained.

| Version | Supported          |
| ------- | ------------------ |
| Latest (main branch) | :white_check_mark: |
| Older commits | :x: |

## Reporting a Vulnerability

### Private Reporting (Preferred)

If you discover a security vulnerability in my repository's code, scripts, or documentation infrastructure, please report it **privately**:

**📧 Email:** gskhalsa6245@gmail.com

**Subject Format:** `[SECURITY] Brief description of vulnerability`

### What to Include in Your Report

To help me address the issue effectively, please provide:

1. **Vulnerability Description**
   - Clear explanation of the security issue
   - Affected component (script, documentation, configuration, etc.)

2. **Reproduction Steps**
   - Detailed steps to reproduce the vulnerability
   - Required conditions or prerequisites

3. **Impact Assessment**
   - Potential security impact
   - Who or what could be affected

4. **Proof of Concept** (if applicable)
   - Code snippets, screenshots, or demonstrations
   - Please be responsible - do not exploit maliciously

5. **Suggested Remediation** (optional)
   - Your recommendations for fixing the issue
   - References to security best practices

### Example Report
```
Subject: [SECURITY] Python script exposes sensitive credentials

Description: 
The authentication helper script (auth_helper.py) logs plaintext 
passwords to console output when DEBUG mode is enabled.

Steps to Reproduce:
1. Enable DEBUG=True in config
2. Run auth_helper.py with test credentials
3. Observe plaintext password in console logs

Impact:
Credentials could be exposed in CI/CD logs or screen recordings.

Suggested Fix:
Mask sensitive data in logging output using '***' redaction.
```

## Response Timeline

I'm committed to addressing security issues promptly:

| Stage | Timeline |
|-------|----------|
| **Initial Acknowledgment** | Within 48-72 hours |
| **Preliminary Assessment** | Within 5-7 days |
| **Status Update** | Weekly until resolved |
| **Resolution & Disclosure** | Varies by severity (see below) |

### Severity Levels

- 🔴 **Critical:** Immediate exposure of credentials, RCE in scripts → 1-3 days
- 🟠 **High:** Significant security flaw in automation tools → 1-2 weeks
- 🟡 **Medium:** Documentation errors that could mislead → 2-4 weeks
- 🟢 **Low:** Minor issues with minimal impact → 4-8 weeks

## Responsible Disclosure

I appreciate security researchers who:

✅ Allow reasonable time to fix issues before public disclosure  
✅ Avoid accessing, modifying, or deleting data without authorization  
✅ Do not exploit vulnerabilities for malicious purposes  
✅ Follow ethical hacking principles and responsible disclosure practices  
✅ Respect the educational nature of this project  

### Coordinated Disclosure

- I prefer **90-day coordinated disclosure** for serious vulnerabilities
- I will credit researchers (if desired) in security advisories
- Public disclosure only after fixes are implemented and tested

## What I Commit To

When you report a vulnerability responsibly, I commit to:

1. **Acknowledge your report** within 72 hours
2. **Keep you informed** of my progress toward resolution
3. **Credit you publicly** (if you wish) when the issue is resolved
4. **Not pursue legal action** against good-faith security researchers

## Out of Scope

The following are **NOT** considered security vulnerabilities in this project:

- ❌ Vulnerabilities that are the **intended subject** of educational labs
- ❌ Issues in third-party platforms (PortSwigger, training environments, etc.)
- ❌ Social engineering attempts against me
- ❌ Denial of service attacks against GitHub infrastructure
- ❌ Reports from automated scanners without manual validation
- ❌ Issues already known and documented

## Important Notes for Educational Content

### For Lab Vulnerabilities

This repository **intentionally documents** security vulnerabilities as part of its educational mission. If you're reporting a vulnerability that is:

- ✅ **Part of a documented lab** → Not a security issue, this is educational content
- ❌ **In my automation scripts/tools** → Valid security concern, please report
- ❌ **In repository infrastructure** → Valid security concern, please report

### Ethical Use Policy

All content in this repository is provided for **authorized educational purposes only**:

- ⚖️ Only test on systems you own or have explicit permission to test
- ⚖️ Respect all applicable laws and regulations
- ⚖️ Use knowledge gained responsibly and ethically

## Contact

**Security Contact:** gskhalsa6245@gmail.com  
**Project Maintainer:** Gurpreet Singh  
**GitHub:** [@Gurpreet-Singh-offensive-Security](https://github.com/Gurpreet-Singh-offensive-Security)

## Additional Resources

- [GitHub Security Advisories Documentation](https://docs.github.com/en/code-security/security-advisories)
- [OWASP Vulnerability Disclosure Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html)
- [Responsible Disclosure Guidelines](https://www.bugcrowd.com/resources/glossary/responsible-disclosure/)

---

**Last Updated:** January 2026  
**Version:** 1.0

---

<div align="center">

### 🛡️ Security is a Journey, Not a Destination

*Thank you for helping keep Offensive Security Labs secure and trustworthy for the community!*

**Gurpreet Singh** | Offensive Security Researcher  
Making security education accessible and safe

</div>
