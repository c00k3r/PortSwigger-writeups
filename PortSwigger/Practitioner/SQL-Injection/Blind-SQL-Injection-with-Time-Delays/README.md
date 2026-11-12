# SQL Injection: Blind SQL Injection with Time Delays and Information Retrieval

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – Blind SQL Injection (Time-Based)

## Lab Objective
Exploit a blind SQL injection vulnerability in the application's tracking cookie to retrieve the administrator's password using conditional time delays.

The objective is to extract sensitive database information by observing differences in HTTP response times rather than relying on visible query results or error messages.

## Vulnerability Overview
Blind SQL injection occurs when an application is vulnerable to SQL injection but does not directly return the results of injected queries.

In this lab, the application uses a tracking cookie for analytics and incorporates its value into an SQL query without proper parameterization.

Since the application does not reveal database query results or provide distinguishable error messages, time-based SQL injection can be used to extract information.

This technique involves injecting conditional SQL statements that cause the database to pause execution when a particular condition is true.

By observing the response time, an attacker can determine whether the tested condition was satisfied.

Repeating this process for individual character positions allows the attacker to reconstruct sensitive information, such as the administrator's password.

## Methodology
1. Reconnaissance of application cookies and HTTP requests.
2. Identifying the `TrackingId` cookie as the vulnerable input.
3. Intercepting requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Testing SQL injection using time-delay functions.
6. Observing response-time differences.
7. Constructing conditional time-delay payloads.
8. Extracting the administrator's password character by character.

## Exploitation Steps
1. Open the lab and browse the application.
2. Launch Burp Suite and navigate to Proxy → HTTP history.
3. Locate a request containing the `TrackingId` cookie.
4. Send the request to Burp Suite Repeater.
5. Modify the tracking cookie to test whether SQL injection is possible.
6. Use a database-specific time-delay function to introduce a noticeable delay.
7. Observe the HTTP response time to confirm whether the injected SQL statement executes.
8. Construct a conditional SQL payload that introduces a delay only when a specified condition is true.
9. Test individual characters of the administrator's password using different character positions and values.
10. Observe response times to determine which conditions are satisfied.
11. Repeat the process until the complete password is extracted.
12. Log in using the administrator credentials and verify that the lab is solved.

## Payloads

**Testing a time delay (PostgreSQL):**
```sql
TrackingId=x'||pg_sleep(5)--
```

**Conditional time delay:**
```sql
TrackingId=x'||(SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END)--
```

**Checking a specific administrator password character:**
```sql
TrackingId=x'||(SELECT CASE WHEN SUBSTRING(password,1,1)='a' THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users WHERE username='administrator')--
```

**Note:** The character position and tested character must be changed during extraction. The payload syntax may need adjustment depending on the database and the original SQL query context.

## Screenshots
<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause
The vulnerability exists because the application directly incorporates an untrusted tracking cookie into dynamically constructed SQL queries.

The absence of parameterized queries allows attackers to inject SQL statements and execute database functions.

Although the application does not expose query results or database errors, differences in query execution time create a side channel that allows attackers to infer sensitive information.

## Remediation
- Use parameterized queries and prepared statements.
- Never concatenate untrusted cookie values directly into SQL statements.
- Implement strict server-side input validation.
- Avoid exposing database execution details through application behavior.
- Apply the principle of least privilege to database accounts.
- Monitor unusual database execution times and suspicious requests.
- Restrict access to sensitive database tables and columns.

## References
- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind)
- [PortSwigger – Time-Based Blind SQL Injection](https://portswigger.net/web-security/sql-injection/blind#time-delays)