# 🛡️ Offensive Security Labs

**Mission:** Master web application security through systematic, hands-on exploitation of 150+ practical labs. Every vulnerability documented with detailed writeups, screenshots, and professional analysis.

## 📋 About This Repository

This repository chronicles my journey to becoming a Red Team Engineer through comprehensive documentation of offensive security techniques. Each lab represents real-world vulnerabilities with step-by-step exploitation methodology, mitigation strategies, and lessons learned.

**Learning Philosophy:** Quality documentation over quantity. Every lab includes detailed analysis, screenshots, and professional security writeups.

## 🎯 Current Progress

### Overall Statistics

Total Labs Available:  278
Labs Completed:        0   (0%)
Current Focus:         Clickjacking & Client-Side Attacks
Next Up:              Authentication Vulnerabilities

### Currently Working On 🔄

| Topic | Progress | Status | Priority |
|-------|----------|--------|----------|
| **Clickjacking (Web Deception)** | 0/5 | 🔥 In Progress | High |

---

## 📂 Repository Structure

Offensive-Security-Labs/
│
├── Server-Side-Vulnerabilities/
│   ├── 01-SQL-Injection/              [0/18 labs]
│   ├── 02-Authentication/             [0/14 labs]  🎯 NEXT
│   ├── 03-Path-Traversal/             [0/6 labs]
│   ├── 04-Access-Control/            [0/13 labs]
│   ├── 05-Information-Disclosure/     [0/5 labs]
│   ├── 06-Business-Logic/            [0/11 labs]
│   ├── 07-Command-Injection/          [0/5 labs]
│   ├── 08-SSRF/                       [0/7 labs]
│   ├── 09-XXE-Injection/              [0/9 labs]
│   ├── 10-NoSQL-Injection/            [0/4 labs]
│   ├── 11-API-Testing/                [0/5 labs]
│   ├── 12-File-Upload/                [0/7 labs]
│   ├── 13-Race-Conditions/            [0/6 labs]
│   └── 14-Web-Cache-Deception/        [0/5 labs]
│
├── Client-Side-Vulnerabilities/
│   ├── 01-XSS/                       [0/30 labs]
│   ├── 02-CSRF/                      [0/12 labs]
│   ├── 03-Clickjacking/               [0/5 labs]  🔥 NOW
│   ├── 04-CORS/                       [0/3 labs]
│   ├── 05-DOM-Based/                  [0/7 labs]
│   └── 06-WebSockets/                 [0/3 labs]
│
├── Advanced-Techniques/
│   ├── 01-HTTP-Request-Smuggling/    [0/21 labs]
│   ├── 02-Web-Cache-Poisoning/       [0/13 labs]
│   ├── 03-Insecure-Deserialization/  [0/10 labs]
│   ├── 04-SSTI/                      [0/7 labs]
│   ├── 05-JWT/                        [0/8 labs]
│   ├── 06-OAuth/                      [0/6 labs]
│   ├── 07-Prototype-Pollution/       [0/10 labs]
│   ├── 08-GraphQL/                    [0/5 labs]
│   ├── 09-Host-Header-Attacks/        [0/7 labs]
│   ├── 10-Web-LLM-Attacks/            [0/4 labs]
│   └── 11-Essential-Skills/           [0/2 labs]
│
└── README.md

---

## 🎓 Lab Categories Overview

### 🔴 Server-Side Vulnerabilities (125 labs)

| Category                | Labs | Difficulty | Impact | Status |
|----------|--------------------|---------|--------|
| SQL Injection           | 18 | ⭐⭐⭐ | Critical | Planned |
| Access Control          | 13 | ⭐⭐ | High | Planned |
| Authentication          | 14 | ⭐⭐ | Critical | Next Up |
| Business Logic          | 11 | ⭐⭐⭐ | High | Planned |
| XXE Injection           | 9 | ⭐⭐⭐ | High | Planned |
| SSRF                    | 7 | ⭐⭐⭐ | High | Planned |
| File Upload             | 7 | ⭐⭐ | High | Planned |
| Path Traversal          | 6 | ⭐⭐ | Medium | Planned |
| Race Conditions         | 6 | ⭐⭐⭐ | Medium | Planned |
| Command Injection       | 5 | ⭐⭐⭐ | Critical | Planned |
| Information Disclosure  | 5 | ⭐ | Low-Medium | Planned |
| API Testing             | 5 | ⭐⭐ | Medium | Planned |
| Web Cache Deception     | 5 | ⭐⭐⭐ | Medium | Planned |
| NoSQL Injection         | 4 | ⭐⭐⭐ | High | Planned |

### 🟡 Client-Side Vulnerabilities (60 labs)

| Category                 | Labs | Difficulty | Impact | Status |
|----------|------|--------|---------|--------|
| Cross-Site Scripting (XSS) | 30 | ⭐⭐⭐ | High | Planned |
| CSRF                       | 12 | ⭐⭐ | Medium-High | Planned |
| DOM-Based Vulns            | 7 | ⭐⭐⭐ | Medium-High | Planned |
| **Clickjacking**           | 5 | **⭐** | **Medium** | **🔥 In Progress** |
| CORS                       | 3 | ⭐⭐ | Medium | Planned |
| WebSockets                 | 3 | ⭐⭐ | Medium | Planned |

### 🟣 Advanced Techniques (93 labs)

| Category                 | Labs | Difficulty | Impact | Status |
|--------------------------|------|---------|--------|
| HTTP Request Smuggling   | 21 | ⭐⭐⭐⭐ | Critical | Future |
| Web Cache Poisoning      | 13 | ⭐⭐⭐⭐ | High | Future |
| Insecure Deserialization | 10 | ⭐⭐⭐⭐ | Critical | Future |
| Prototype Pollution      | 10 | ⭐⭐⭐⭐ | High | Future |
| JWT                      | 8 | ⭐⭐⭐ | High | Future |
| SSTI                     | 7 | ⭐⭐⭐⭐ | Critical | Future |
| Host Header Attacks      | 7 | ⭐⭐⭐ | Medium-High | Future |
| OAuth                    | 6 | ⭐⭐⭐ | High | Future |
| GraphQL                  | 5 | ⭐⭐⭐ | Medium-High | Future |
| Web LLM Attacks          | 4 | ⭐⭐⭐⭐ | High | Future |
| Essential Skills         | 2 | ⭐⭐ | N/A | Future |

---

## 🛠️ Tools & Methodology

### Primary Tools

**Exploitation:**
- Burp Suite Professional (Intruder, Repeater, Scanner)
- Kali Linux Updated && Upgraded
- Python latest for automation scripts
- Browser DevTools (Chrome/Firefox)

**Documentation:**
- Markdown for writeups
- GitHub for version control
- Screenshots with annotations

### Lab Methodology

1. RECONNAISSANCE
   └─ Analyze target functionality
   └─ Identify injection points
   └─ Enumerate attack surface

2. VULNERABILITY ANALYSIS
   └─ Test for security flaws
   └─ Understand underlying weakness
   └─ Research exploitation techniques

3. EXPLOITATION
   └─ Craft and test payloads
   └─ Document each step with screenshots
   └─ Achieve lab objective

4. DOCUMENTATION
   └─ Write detailed walkthrough
   └─ Explain key concepts learned
   └─ Document mitigation strategies

5. REFLECTION
   └─ Analyze what worked/didn't work
   └─ Note real-world applications
   └─ Identify areas for deeper study

---

## 📖 Documentation Standards

Every lab writeup includes:

### Required Sections
- **Lab Information** - Platform, difficulty, date completed
- **Objective** - What we're trying to achieve
- **Reconnaissance** - Initial findings and analysis
- **Exploitation Steps** - Detailed walkthrough with screenshots
- **Key Concepts** - Technical explanations
- **Mitigation** - How to prevent the vulnerability
- **References** - Additional learning resources
- **Personal Notes** - Challenges, insights, real-world impact

### Screenshot Requirements
- Minimum 6-8 screenshots per lab
- Clear, annotated images
- Numbered sequentially (01-initial.png, 02-exploit.png, etc.)
- Show both process and results

---

## 📊 Learning Roadmap

### Phase 1: Client-Side Fundamentals (Dec 2025 - Jan 2026)
🔥 Clickjacking              [0/5]   ← Starting Here
🎯 Authentication           [0/14]   ← Next
   XSS                      [0/30]
   CSRF                     [0/12]
   DOM-Based                 [0/7]

### Phase 2: Server-Side Core (Feb - Apr 2026)
   
   SQL Injection            [0/18]
   Path Traversal            [0/6]
   Command Injection         [0/5]
   SSRF                      [0/7]
   XXE                       [0/9]
### Phase 3: Application Logic (May - Jul 2026)

   Access Control          [0/13]
   Business Logic          [0/11]
   File Upload              [0/7]
   Information Disclosure   [0/5]
### Phase 4: Advanced Topics (Aug 2026 - 2027)

   Request Smuggling       [0/21]
   Deserialization        [0/10]
   Cache Poisoning        [0/13]
   SSTI                    [0/7]
   JWT & OAuth            [0/14]
   Prototype Pollution    [0/10]

## 📅 Current Focus: Clickjacking Labs

### 🎯 Next 5 Labs (This Week)

1. **Lab 1:** Basic clickjacking with CSRF token protection
   - Status: 📝 Planning
   - Difficulty: Apprentice
   - Estimated Time: 1 hour

2. **Lab 2:** Clickjacking with form input data prefilled
   - Status: ⏳ Queued
   - Difficulty: Apprentice
   - Estimated Time: 1 hour

3. **Lab 3:** Clickjacking with frame buster script
   - Status: ⏳ Queued
   - Difficulty: Apprentice
   - Estimated Time: 1.5 hours

4. **Lab 4:** Exploiting clickjacking to trigger DOM XSS
   - Status: ⏳ Queued
   - Difficulty: Practitioner
   - Estimated Time: 2 hours

5. **Lab 5:** Multistep clickjacking
   - Status: ⏳ Queued
   - Difficulty: Practitioner
   - Estimated Time: 2 hours

**Total Estimated Time:** 7.5 hours for complete Clickjacking topic

---

## 📊 Progress Tracking

### By Difficulty Level

APPRENTICE     ░░░░░░░░░░░░░░░░░░░░   0/89  (0%)
PRACTITIONER   ░░░░░░░░░░░░░░░░░░░░   0/142 (0%)
EXPERT         ░░░░░░░░░░░░░░░░░░░░   0/47  (0%)

### Weekly Goals

**Week 1 (Dec 15-21, 2025):**
- 🎯 Complete Clickjacking (5 labs)

**Week 2 (Dec 22-28, 2025):**

🎯 Start Authentication (5 labs)

**Week 3-4 (Dec 29 - Jan 11, 2026):**

- 🎯 Complete Authentication (remaining 9 labs)

---

## 💡 Learning Goals

### Technical Skills
- Master Burp Suite Professional workflows
- Develop systematic exploitation methodology
- Build comprehensive documentation habits
- Understand vulnerability root causes
- Learn effective mitigation strategies

### Professional Development
- Create portfolio-quality writeups
- Demonstrate hands-on security expertise
- Build GitHub presence
- Prepare for Security+ certification (2027)
- Foundation for OSCP preparation

---

## 🎯 2027 Milestone

**Target by End of 2027:**
- ✅ 150+ labs completed with full documentation
- ✅ All Apprentice & Practitioner labs finished
- ✅ 50%+ Expert labs completed
- ✅ Security+ certified
- ✅ Professional portfolio demonstrating real-world skills

---

## 📚 Resources & References

### Primary Learning Platform
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)

### Additional Resources
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [HackerOne Hacktivity](https://hackerone.com/hacktivity)
- [Bug Bounty Writeups](https://pentester.land/list-of-bug-bounty-writeups.html)

### Tools Documentation
- [Burp Suite Documentation](https://portswigger.net/burp/documentation)
- [OWASP Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

---

## 🔗 Connect

Building toward a career in Red Team operations. Open to learning from experienced security professionals.

**Profile:** [github.com/gurpreet-singh-offensive-security](https://github.com/gurpreet-singh-offensive-security)  
**Email:** sas487f@gmail.com

---

## 📝 Repository Updates

**Last Updated:** December 15, 2025  
**Status:** Just getting started! First labs coming soon.  
**Next Update:** After completing first 5 Clickjacking labs

---

**🚀 Journey to 150+ Labs Starts Now 🚀**
