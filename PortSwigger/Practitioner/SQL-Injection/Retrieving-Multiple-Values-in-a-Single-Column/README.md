# SQL Injection UNION Attack, Retrieving Multiple Values in a Single Column

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – UNION-Based SQL Injection

## Lab Objective
Exploit a SQL injection vulnerability in the product category filter to retrieve multiple values from the `users` table, despite the application displaying only a single column in its response.

The goal is to extract usernames and passwords using SQL concatenation.

## Vulnerability Overview
This lab demonstrates a UNION-based SQL injection vulnerability in the product category filter.

The application incorporates user-controlled input into SQL queries without proper parameterization, allowing an attacker to inject additional SQL statements using the UNION operator.

A limitation arises when the application only displays one column in its response. To overcome this, SQL concatenation can combine multiple database fields into a single string.

The `||` operator is used to concatenate the `username` and `password` columns, while the `~` character acts as a separator between the values.

This allows an attacker to retrieve multiple database values through a single visible column.

## Methodology
1. Reconnaissance of the product category filtering functionality.
2. Identifying the vulnerable `category` parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Determining the number of columns returned by the original SQL query.
6. Identifying a column that accepts text values.
7. Constructing a UNION SELECT statement.
8. Using SQL concatenation to retrieve multiple database values through one column.

## Exploitation Steps
1. Navigate to the product listing page and select any product category.
2. Intercept the resulting GET request using Burp Suite Proxy.
3. Send the request to Burp Suite Repeater.
4. Determine the number of columns returned by the original query using UNION SELECT NULL statements.
5. Identify which column accepts text values by injecting a recognizable string.
6. Construct a UNION SELECT payload that combines the `username` and `password` fields.
7. Use the `||` operator to concatenate both fields, with `~` as a separator.
8. Send the modified request through Repeater.
9. Inspect the HTTP response to identify the retrieved usernames and passwords.
10. Identify the administrator account credentials and verify successful exploitation.

## Payloads

**Determining the number of columns:**
```sql
' UNION SELECT NULL,NULL--
```

**Identifying a text-compatible column:**
```sql
' UNION SELECT 'test',NULL--
```

**Final exploitation payload:**
```sql
' UNION SELECT username || '~' || password FROM users--
```

## Screenshots
<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause
The vulnerability exists because the application directly incorporates untrusted user input into dynamically constructed SQL queries.

The absence of parameterized queries allows attackers to manipulate the original SQL statement and append additional queries using the UNION operator.

Additionally, displaying database query results directly in the application response exposes sensitive information from other database tables.

## Remediation
- Use parameterized queries and prepared statements.
- Avoid concatenating user-controlled input directly into SQL statements.
- Implement strict server-side input validation.
- Apply the principle of least privilege to database accounts.
- Restrict access to sensitive database tables and columns.
- Store passwords using secure hashing algorithms such as Argon2id or bcrypt.
- Avoid exposing sensitive database information in application responses.

## References
- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [SQL Injection UNION Attacks](https://portswigger.net/web-security/sql-injection/union-attacks)