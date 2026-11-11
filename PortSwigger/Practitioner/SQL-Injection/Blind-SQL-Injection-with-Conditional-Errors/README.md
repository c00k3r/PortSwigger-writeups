# Blind SQL Injection with Conditional Errors

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – Blind SQL Injection (Conditional Error-Based)

## Lab Objective
Exploit a blind SQL injection vulnerability in the tracking cookie used by the application to retrieve the administrator's password from the `users` table.

The objective is to use conditional SQL errors to extract sensitive information character by character, even though the application does not directly display database query results.

## Vulnerability Overview
Blind SQL injection occurs when an application is vulnerable to SQL injection but does not return the results of the injected SQL query directly in its HTTP response.

In this lab, the application uses a tracking cookie for analytics and incorporates its value into an SQL query without proper parameterization.

Although the application does not respond differently based on whether the query returns rows, it returns a custom error message when the SQL query causes an error.

This behavior can be exploited using conditional SQL expressions. By constructing queries that deliberately generate an error when a specific condition is true, an attacker can infer information from the database.

Using this technique, individual characters of the administrator's password can be extracted by testing different character positions and values.

## Methodology
1. Reconnaissance of application cookies and HTTP requests.
2. Identifying the tracking cookie as the vulnerable input.
3. Intercepting requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Testing SQL injection using conditional expressions.
6. Identifying the application's custom error response.
7. Extracting database information character by character.
8. Retrieving the administrator's password.

## Exploitation Steps
1. Open the lab and browse the application.
2. Open Burp Suite and navigate to Proxy → HTTP history.
3. Locate a request containing the `TrackingId` cookie.
4. Send the request to Burp Suite Repeater.
5. Modify the `TrackingId` cookie value to test whether SQL injection is possible.
6. Observe the application's response and identify the custom error behavior.
7. Construct a conditional SQL payload that generates a database error when a specified condition is true.
8. Use a conditional expression to check whether the administrator's password matches a particular character at a specific position.
9. Observe whether the application returns the custom error message to determine the result of each condition.
10. Repeat the process for successive character positions to reconstruct the administrator's password.
11. Use the extracted password to log in as the administrator and verify that the lab is solved.

## Payloads

**Testing conditional SQL errors:**
```sql
TrackingId=xyz'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'
```

**Checking a specific password character:**
```sql
TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'
```

**Checking subsequent characters:**

Change the character position and tested character in the payload to progressively extract the password.

## Screenshots
<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause
The root cause is the application's failure to properly parameterize the tracking cookie value before incorporating it into an SQL query.

This allows attackers to inject conditional SQL expressions into the database query.

Additionally, the application's custom error handling creates an observable difference between successful and failed conditions, allowing attackers to infer sensitive database information without directly viewing query results.

## Remediation
- Use parameterized queries and prepared statements.
- Never concatenate untrusted cookie values directly into SQL statements.
- Implement strict server-side input validation.
- Avoid exposing database errors or distinguishable error responses.
- Use generic error messages for database failures.
- Apply the principle of least privilege to database accounts.
- Monitor suspicious requests containing SQL operators and database functions.
- Restrict access to sensitive database information.

## References
- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind)