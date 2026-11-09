# SQL Injection UNION Attack, Determining the Number of Columns Returned by the Query

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – UNION-Based SQL Injection

## Lab Objective
Exploit a SQL injection vulnerability in the product category filter to determine the number of columns returned by the application's underlying SQL query.

The objective is to identify the correct number of columns so that a UNION-based SQL injection attack can be constructed successfully.

## Vulnerability Overview
SQL Injection occurs when an application incorporates untrusted user input into SQL queries without proper validation or parameterization.

In this lab, the product category filter is vulnerable to SQL injection. The application uses the supplied category parameter in a backend SQL query and returns the query results in its HTTP response.

A UNION attack allows an attacker to append the results of another SELECT statement to the original query.

However, for a UNION query to execute successfully, both SELECT statements must return the same number of columns.

The number of columns can be identified using two common techniques:

1. **ORDER BY Method:** Increment the column index until the database returns an error. The highest valid index indicates the number of columns.
2. **UNION SELECT NULL Method:** Inject increasing numbers of NULL values until the application accepts the UNION query without producing an error.

NULL values are useful because they can be converted to most common SQL data types, making them suitable for identifying the required column count.

## Methodology
1. Reconnaissance of the application and product category filtering functionality.
2. Identifying the vulnerable category parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Testing SQL injection using single quotes and observing application responses.
6. Determining the number of columns using ORDER BY statements.
7. Verifying the column count using UNION SELECT with NULL values.
8. Identifying a valid UNION query structure for further exploitation.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Select any product category and observe how the application filters products.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate the GET request responsible for filtering products. Identify the `category` parameter.

5. Send the request to Burp Suite Repeater.

6. Test the parameter by appending a single quote (`'`) to the existing category value.

7. Observe the server response. An SQL error or unexpected response may indicate that the input is being incorporated into a backend SQL query.

8. Begin determining the number of columns using the ORDER BY technique.

9. Append the following payload to the category parameter:

   `' ORDER BY 1--`

10. Increment the column index:

   - `' ORDER BY 1--`
   - `' ORDER BY 2--`
   - `' ORDER BY 3--`
   - Continue until the application returns an error.

11. The highest valid column index indicates the number of columns returned by the original query.

12. Alternatively, verify the result using UNION SELECT statements containing increasing numbers of NULL values.

13. For example, if the original query returns three columns, use:

   `' UNION SELECT NULL,NULL,NULL--`

14. Send the request using Burp Repeater and inspect the response.

15. If the query executes without an SQL error, the supplied UNION SELECT statement has the correct number of columns.

16. Return to the lab and verify that it is marked as solved.

## Payloads

**Original category request (illustrative):**

```http
GET /filter?category=Gifts HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Initial SQL injection test:**

```sql
'
```

**ORDER BY technique:**

```sql
' ORDER BY 1--
```

```sql
' ORDER BY 2--
```

```sql
' ORDER BY 3--
```

```sql
' ORDER BY 4--
```

**UNION SELECT technique (example for three columns):**

```sql
' UNION SELECT NULL,NULL,NULL--
```

**Example HTTP request:**

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,NULL,NULL-- HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Note:** The number of NULL values must match the actual number of columns returned by the original query. The example above assumes three columns. The SQL comment syntax may need adjustment depending on the database.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application directly incorporates user-controlled input into a dynamically constructed SQL query.

The application does not properly separate SQL instructions from user-supplied data.

As a result, an attacker can manipulate the structure of the original SQL statement and introduce additional SQL operations through the UNION operator.

The application also exposes query results and potentially database errors through its responses, allowing attackers to determine the structure of the underlying query.

## Remediation

- Use parameterized queries and prepared statements.
- Never concatenate untrusted user input directly into SQL queries.
- Implement strict input validation using allow-lists for category parameters.
- Apply least-privilege permissions to database accounts.
- Avoid exposing detailed database errors to users.
- Use secure database access libraries and ORM frameworks where appropriate.
- Implement proper exception handling and server-side logging.

## References

- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – SQL Injection UNION Attacks](https://portswigger.net/web-security/sql-injection/union-attacks)
- [PortSwigger – Determining the Number of Columns](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)