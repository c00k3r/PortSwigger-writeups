# SQL Injection UNION Attack, Finding a Column Containing Text

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection (SQLi) – UNION-Based SQL Injection

## Lab Objective
Exploit a SQL injection vulnerability in the product category filter to identify a column in the query results that can store and display text data.

The objective is to determine which column accepts string values so that it can be used to retrieve information from other database tables through a UNION-based SQL injection attack.

## Vulnerability Overview
SQL Injection occurs when an application incorporates untrusted user input into SQL queries without properly separating SQL instructions from user-supplied data.

In this lab, the product category filter is vulnerable to SQL injection. The application returns the results of the underlying SQL query in its HTTP response, allowing an attacker to inject additional SQL statements using the UNION operator.

After determining the number of columns returned by the original query, the next step is to identify which columns accept text data.

Different database columns can have different data types, such as integers, strings, dates, and other values.

A UNION query requires both SELECT statements to return the same number of columns, and corresponding columns must have compatible data types.

To identify a text-compatible column, an attacker can inject a UNION SELECT statement containing a recognizable string value in one column while using NULL values in the remaining columns.

By moving the string between column positions and observing the application response, it is possible to determine which column accepts and displays text.

## Methodology
1. Reconnaissance of the application and product category filtering functionality.
2. Identifying the vulnerable category parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the request to Burp Suite Repeater.
5. Confirming the SQL injection vulnerability.
6. Determining the number of columns returned by the original query.
7. Injecting recognizable string values into individual columns.
8. Observing application responses to identify a text-compatible column.
9. Confirming the successful UNION-based injection.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Select any product category and observe how the application filters products.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate the GET request responsible for filtering products. Identify the `category` parameter.

5. Send the request to Burp Suite Repeater.

6. Use the technique from the previous lab to determine the number of columns returned by the original SQL query.

7. Once the column count is known, construct a UNION SELECT statement containing NULL values in all positions.

8. Replace one NULL value at a time with a recognizable string, such as `test`.

9. For example, if the original query returns three columns, test:

   `' UNION SELECT 'test',NULL,NULL--`

10. Send the request and inspect the response.

11. If the application returns an SQL error, the tested column may not accept string data.

12. Move the string value to the next column:

   `' UNION SELECT NULL,'test',NULL--`

13. Continue testing the remaining columns until the application successfully displays the injected string.

14. Identify the column position where the text value is accepted and reflected in the response.

15. This confirms the presence of a text-compatible column that can be used in subsequent UNION-based SQL injection attacks.

16. Return to the lab and verify that it is marked as solved.

## Payloads

**Original category request (illustrative):**

```http
GET /filter?category=Gifts HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Initial UNION SELECT statement (three-column example):**

```sql
' UNION SELECT NULL,NULL,NULL--
```

**Testing the first column:**

```sql
' UNION SELECT 'test',NULL,NULL--
```

**Testing the second column:**

```sql
' UNION SELECT NULL,'test',NULL--
```

**Testing the third column:**

```sql
' UNION SELECT NULL,NULL,'test'--
```

**Example HTTP request:**

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,'test',NULL-- HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Note:** The examples assume the original query returns three columns. Adjust the number of NULL values according to the actual column count identified in the previous lab. The position of the string must be determined experimentally.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application directly incorporates user-controlled input into dynamically constructed SQL queries.

The absence of parameterized queries allows attackers to manipulate the original SQL statement and introduce additional SELECT operations through the UNION operator.

Furthermore, the application exposes query results in its HTTP response, allowing injected data to become visible to an attacker.

The underlying issue is insufficient separation between SQL commands and untrusted user input.

## Remediation

- Use parameterized queries and prepared statements.
- Never concatenate untrusted user input directly into SQL queries.
- Implement strict input validation for category parameters.
- Apply least-privilege permissions to database accounts.
- Avoid exposing detailed database errors to users.
- Implement proper exception handling and server-side logging.
- Use secure database access libraries and ORM frameworks where appropriate.

## References

- [PortSwigger Web Security Academy – SQL Injection](https://portswigger.net/web-security/sql-injection)
- [PortSwigger – SQL Injection UNION Attacks](https://portswigger.net/web-security/sql-injection/union-attacks)
- [PortSwigger – Finding a Column Containing Text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)