# PortSwigger Web Security Academy — Lab Writeups

A collection of hands-on lab writeups from [PortSwigger Web Security Academy](https://portswigger.net/web-security), documenting my journey into web application security and penetration testing.

This repository contains technical notes, vulnerability analysis, exploitation methodologies, and step-by-step solutions to labs covering common web application vulnerabilities.

The goal is to develop a deeper understanding of web security by learning how vulnerabilities work, how they can be exploited in controlled environments, and how they can be prevented.

## Topics Covered

* SQL Injection (SQLi)
* Cross-Site Scripting (XSS)
* Cross-Site Request Forgery (CSRF)
* Cross-Origin Resource Sharing (CORS)
* Cross-Site Request Forgery (CSRF)
* Clickjacking
* DOM-Based Vulnerabilities
* XML External Entity (XXE) Injection
* Server-Side Request Forgery (SSRF)
* HTTP Request Smuggling
* OS Command Injection
* Server-Side Template Injection (SSTI)
* Path Traversal
* Access Control Vulnerabilities
* Authentication Vulnerabilities
* Business Logic Vulnerabilities
* Information Disclosure
* File Upload Vulnerabilities
* Race Conditions
* WebSockets Security
* API Testing
* JWT Attacks
* OAuth Authentication Vulnerabilities
* Essential Skills and Miscellaneous Topics

*Topics will be added as I progress through the academy.*

## Repository Structure

```text
portswigger-lab-writeups/
├── SQL-Injection/
│   ├── lab-01-description.md
│   └── lab-02-description.md
├── Cross-Site-Scripting/
│   ├── lab-01-description.md
│   └── lab-02-description.md
├── Authentication/
│   └── lab-01-description.md
├── Access-Control/
│   └── lab-01-description.md
├── SSRF/
│   └── lab-01-description.md
├── XXE/
│   └── lab-01-description.md
└── README.md
```

The directory structure may evolve as more labs are completed.

## Writeup Format

Each writeup aims to document the vulnerability and its exploitation process in a clear, reproducible format.

Depending on the lab, a writeup may include:

1. **Lab Overview** — Lab title, vulnerability category, and difficulty.
2. **Objective** — What needs to be achieved to solve the lab.
3. **Vulnerability Analysis** — The underlying security flaw and why it exists.
4. **Methodology** — The approach used to investigate the application.
5. **Exploitation Steps** — Step-by-step explanation of the solution.
6. **Payloads and Requests** — Relevant payloads, HTTP requests, and responses.
7. **Root Cause** — The technical reason the vulnerability occurs.
8. **Mitigation** — Recommended defensive measures.
9. **Key Takeaways** — Important lessons learned from the lab.

Not every writeup will contain every section; the format will depend on the vulnerability being explored.

## Tools & Technologies

Tools and technologies used throughout this journey may include:

* Burp Suite
* Browser Developer Tools
* HTTP and HTTPS
* JavaScript
* SQL
* Python
* Linux
* Command-line utilities

## Learning Objectives

* Develop a strong understanding of common web application vulnerabilities.
* Improve practical web penetration testing and vulnerability assessment skills.
* Learn to analyze HTTP requests and responses effectively.
* Understand how to identify, exploit, and validate security flaws.
* Explore vulnerability root causes and effective remediation techniques.
* Build a structured, publicly accessible knowledge base of hands-on security research.
* Strengthen my technical documentation and reporting skills.

## Progress Tracking

Lab solutions and writeups will be added progressively as I work through the Web Security Academy.

| Category                       | Status      |
| ------------------------------ | ----------- |
| SQL Injection                  | In Progress |
| Cross-Site Scripting           | In Progress |
| Authentication                 | In Progress |
| Access Control                 | In Progress |
| Server-Side Request Forgery    | In Progress |
| XML External Entity Injection  | In Progress |
| Other Vulnerability Categories | Upcoming    |

*Progress statuses will be updated as the repository grows.*

## Disclaimer

This repository is intended for educational purposes and authorized security testing.

All practical exercises documented here are based on labs provided by PortSwigger Web Security Academy or other explicitly authorized environments.

The techniques discussed should only be used on systems for which you have explicit permission to conduct security testing.

## Acknowledgements

* [PortSwigger Web Security Academy](https://portswigger.net/web-security) — For providing free, hands-on web security training and interactive vulnerability labs.
* [PortSwigger](https://portswigger.net/) — For developing Burp Suite and supporting web application security education.

---

**Learning by doing. Understanding vulnerabilities. Building better security.**
