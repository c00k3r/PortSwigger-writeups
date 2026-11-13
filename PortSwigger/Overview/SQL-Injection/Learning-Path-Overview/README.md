# SQL Injection (SQLi) – Complete Overview

## 1. Introduction

SQL Injection (SQLi) is a web application vulnerability that occurs when an application incorporates untrusted user input into SQL queries without properly handling or parameterizing that input.

An attacker can manipulate the application's database queries to interfere with their intended execution.

Depending on the vulnerability and database permissions, SQL injection can allow attackers to:

- Bypass authentication mechanisms.
- Retrieve sensitive information from database tables.
- Access usernames, passwords and other confidential records.
- Modify or delete database records.
- Extract database metadata and schema information.
- In certain configurations, execute administrative database operations or interact with the underlying operating system.

SQL injection is primarily caused by insecure construction of database queries.

---

## 2. Understanding SQL

SQL stands for Structured Query Language. It is used to communicate with relational database management systems (RDBMS).

Common database systems include:

- MySQL
- PostgreSQL
- Microsoft SQL Server
- Oracle Database
- SQLite

### Common SQL Commands

| Command | Description |
|---|---|
| SELECT | Retrieves records from a database |
| INSERT | Inserts new records |
| UPDATE | Modifies existing records |
| DELETE | Deletes records |
| CREATE | Creates database objects |
| DROP | Deletes database objects |
| ALTER | Modifies database structures |
| WHERE | Filters records |
| UNION | Combines results from multiple SELECT queries |
| ORDER BY | Sorts query results |

### Example Database

Consider a database containing a table named `users`.

| id | username | password | role |
|---|---|---|---|
| 1 | admin | admin123 | administrator |
| 2 | john | john123 | user |
| 3 | alice | alice123 | user |

A normal SQL query:

```sql
SELECT * FROM users WHERE username = 'admin';
```

This retrieves the record belonging to the administrator.

SQL injection becomes possible when an application allows user-controlled input to modify the structure or logic of such queries.

---

## 3. How SQL Injection Works

Consider a login functionality that accepts a username and password.

The backend constructs the following query:

```sql
SELECT * FROM users
WHERE username = 'admin'
AND password = 'password123';
```

If the application directly concatenates user input into this query, an attacker may manipulate its structure.

For example, entering the following username:

```sql
admin'--
```

Could result in:

```sql
SELECT * FROM users
WHERE username = 'admin'--'
AND password = 'anything';
```

The `--` sequence starts a SQL comment in many database systems, causing the remaining part of the query to be ignored.

If the query executes successfully, authentication may be bypassed.

### Important SQL Injection Characters and Operators

| Character / Operator | Purpose |
|---|---|
| `'` | Terminates a string literal |
| `"` | Can delimit identifiers or string literals, depending on the database |
| `--` | Starts a single-line comment in supported SQL dialects |
| `#` | Single-line comment in MySQL |
| `/* */` | Block comment |
| `OR` | Logical OR operator |
| `AND` | Logical AND operator |
| `UNION` | Combines query results |
| `||` | String concatenation in PostgreSQL and Oracle, among others |
| `+` | String concatenation in SQL Server |

The exact behavior of these characters and operators depends on the database system.

---

# 4. Types of SQL Injection

SQL injection vulnerabilities are commonly classified into three major categories.

## 4.1 In-Band SQL Injection

In-band SQL injection occurs when the attacker uses the same communication channel to inject SQL and retrieve the results.

It is generally easier to identify because the application's response directly exposes the results of the injected query.

Two important techniques are:

### A. Error-Based SQL Injection

Error-based SQL injection involves triggering database errors to obtain information about the database or its structure.

Database errors may disclose:

- Database version.
- Database type.
- Table names.
- Column names.
- SQL query structure.

For example, an incorrectly constructed query might produce an error revealing the database engine or the location of a syntax error.

**Objective:** Use database errors as an information-disclosure channel.

### B. UNION-Based SQL Injection

UNION-based SQL injection uses the SQL `UNION` operator to retrieve information from additional database tables.

The UNION operator combines the results of multiple SELECT statements.

Example:

```sql
SELECT name, description FROM products
UNION
SELECT username, password FROM users;
```

For a UNION query to work:

1. Both SELECT statements must return the same number of columns.
2. Corresponding columns must have compatible data types.
3. The injected results must be displayed by the application.

### Important UNION Attack Concepts

**Determining the number of columns**

The attacker needs to determine how many columns the original query returns.

Common approaches include:

- Using `ORDER BY` with different column indexes.
- Using UNION SELECT statements with different numbers of NULL values.

**Finding columns containing useful data**

Once the column count is known, the attacker identifies which returned columns can display textual information.

**Retrieving database contents**

After identifying compatible columns, an attacker may retrieve information from other tables.

---

## 4.2 Inferential (Blind) SQL Injection

Blind SQL injection occurs when the application does not directly return database query results or useful database errors.

Instead, the attacker observes differences in application behavior to infer information.

Two major techniques are:

### A. Boolean-Based Blind SQL Injection

Boolean-based SQL injection uses conditions that evaluate to TRUE or FALSE.

The attacker observes changes in the application's response.

For example:

- A TRUE condition may return the normal page.
- A FALSE condition may return a different page or no results.

By repeatedly testing conditions, an attacker can infer database information.

This technique can be used to determine:

- Whether a particular database condition is true.
- Whether a table or record exists.
- Individual characters of sensitive values.

### B. Time-Based Blind SQL Injection

Time-based SQL injection uses database delays to infer whether a condition is true.

The attacker introduces a condition that causes the database to wait when a particular condition is satisfied.

The application takes longer to respond when the condition evaluates as expected.

For example:

- Normal response: 200 milliseconds.
- Response with an intentional delay: several seconds.

Repeated observations can reveal information even when the application does not return query results.

---

## 4.3 Out-of-Band SQL Injection

Out-of-band SQL injection uses a separate communication channel to retrieve information from the database.

It is useful when:

- The application does not return query results.
- Boolean-based techniques are difficult.
- Time-based techniques are unreliable.
- The database supports external network interactions.

Depending on the database and its configuration, an attacker may trigger DNS or HTTP requests to an external server.

### Example: DNS-Based Exfiltration

A database may be manipulated into performing a DNS lookup containing attacker-controlled information.

The attacker monitors the DNS interaction using an external service such as Burp Collaborator.

This can reveal that the database processed the injected query and attempted an external interaction.

**Important:** Out-of-band SQL injection depends heavily on database functionality, permissions and network configuration.

---

# 5. SQL Injection Attack Techniques

Apart from the three major categories, several techniques are used to exploit specific SQL injection scenarios.

## 5.1 Authentication Bypass

Authentication bypass attempts to manipulate login queries so that authentication checks do not work as intended.

The most common cause is improper handling of user-controlled input in login queries.

## 5.2 Retrieving Hidden Data

Applications often use SQL queries to filter products, articles or other records.

If a filter parameter is vulnerable, an attacker may manipulate the WHERE clause to retrieve records that should remain hidden.

For example, restricted products or unpublished records may become accessible.

## 5.3 Subverting Application Logic

Some applications use SQL queries to enforce business rules.

SQL injection can manipulate those conditions.

This may allow attackers to interfere with:

- Account verification.
- Authorization checks.
- Product filtering.
- Transaction conditions.
- Other application workflows.

## 5.4 Database Enumeration

Database enumeration involves discovering the structure and contents of a database.

An attacker may attempt to identify:

- Database management system.
- Database version.
- Database names.
- Table names.
- Column names.
- User accounts.
- Password-related information.

### Understanding information_schema

Many relational database systems expose metadata through `information_schema`.

For example:

```sql
SELECT table_name
FROM information_schema.tables;
```

This retrieves table names visible through that metadata view.

To retrieve column names:

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_name = 'users';
```

These queries are particularly useful for understanding how UNION-based SQL injection can expose database structure.

PostgreSQL also provides system catalogs such as:

- `pg_tables`
- `pg_class`
- `pg_attribute`
- `pg_database`

The exact metadata mechanisms differ between database systems.

## 5.5 Retrieving Multiple Values in a Single Column

Sometimes the original query returns only one column.

However, an attacker may need to retrieve multiple values, such as usernames and passwords.

String concatenation can combine multiple values into one output column.

Example for PostgreSQL:

```sql
SELECT username || ':' || password FROM users;
```

Here:

- `||` concatenates strings.
- `:` separates the values.
- Both fields are returned as a single string.

String concatenation syntax differs between database systems.

## 5.6 Filter and WAF Bypass

Web Application Firewalls attempt to identify and block suspicious requests.

However, poorly implemented filtering mechanisms may be bypassed using different representations of the same input.

Common techniques include:

- URL encoding.
- Double URL encoding.
- Case manipulation.
- Comment insertion.
- Whitespace alternatives.
- XML entity encoding.
- Alternative SQL syntax.

### XML Entity Encoding

XML supports character references that represent individual characters.

For example:

| Character | Hexadecimal XML Entity |
|---|---|
| U | `&#x55;` |
| N | `&#x4e;` |
| I | `&#x49;` |
| O | `&#x4f;` |
| S | `&#x53;` |
| E | `&#x45;` |
| L | `&#x4c;` |

An XML parser decodes these references before the application processes the resulting value.

This can bypass simplistic filters that inspect the encoded representation instead of the decoded input.

---

# 6. SQL Injection in Different Database Systems

SQL syntax and functionality vary across database systems.

Understanding these differences is important when performing SQL injection testing.

| Feature | MySQL | PostgreSQL | SQL Server | Oracle |
|---|---|---|---|---|
| String concatenation | `CONCAT()` | `||` | `+` | `||` |
| Single-line comment | `--`, `#` | `--` | `--` | `--` |
| Block comments | `/* */` | `/* */` | `/* */` | `/* */` |
| Database version | `VERSION()` | `version()` | `@@VERSION` | Version views |
| Metadata | `information_schema` | `information_schema`, catalogs | `information_schema` | Data dictionary views |
| Delay functionality | `SLEEP()` | `pg_sleep()` | `WAITFOR DELAY` | `DBMS_LOCK.SLEEP` where permitted |

### Why This Matters

A payload that works against PostgreSQL may not work against MySQL or SQL Server.

During penetration testing, identifying the database technology helps determine which SQL syntax and functions are supported.

---

# 7. Second-Order SQL Injection

Second-order SQL injection is slightly different from conventional SQL injection.

In this case, malicious input is initially stored safely or without being executed.

Later, the application retrieves that stored input and incorporates it into another SQL query without proper parameterization.

The vulnerability occurs during this later operation.

### Example Scenario

1. An application allows users to update their profile information.
2. A malicious value is stored in the database.
3. Another application function retrieves that value.
4. The retrieved value is incorporated into a dynamically constructed SQL query.
5. The SQL injection occurs during this second operation.

**Key distinction:** The input is stored during the first interaction but becomes dangerous when reused in a later SQL query.

---

# 8. SQL Injection Testing Methodology

A structured testing methodology helps identify SQL injection vulnerabilities without relying on random payloads.

### Step 1: Identify Input Parameters

Look for user-controlled input in:

- URL query parameters.
- POST request bodies.
- Login forms.
- Search functionality.
- Product filters.
- Cookies.
- HTTP headers.
- XML and JSON requests.

### Step 2: Intercept Requests

Use Burp Suite Proxy to intercept application requests.

Send interesting requests to Repeater for controlled testing.

### Step 3: Test Input Behavior

Observe how the application responds to different inputs.

Look for:

- SQL errors.
- Unexpected response changes.
- HTTP status code changes.
- Response length differences.
- Response time differences.

### Step 4: Identify the Database

Use error messages, application behavior and database-specific syntax to identify the underlying database technology.

### Step 5: Determine the Injection Type

Determine whether the vulnerability is:

- Error-based.
- UNION-based.
- Boolean-based blind.
- Time-based blind.
- Out-of-band.
- Second-order.

### Step 6: Enumerate Database Structure

Where authorized and applicable, identify:

- Database tables.
- Column names.
- Data types.
- Relevant records.

### Step 7: Demonstrate Impact

Use the minimum necessary proof to demonstrate the vulnerability.

Avoid accessing unrelated sensitive information or modifying production data.

### Step 8: Document Findings

Record:

- Vulnerable endpoint.
- Parameter.
- HTTP request and response.
- Vulnerability type.
- Reproduction steps.
- Security impact.
- Recommended remediation.

---

# 9. SQL Injection Tools

Several tools can assist with SQL injection detection and exploitation during authorized security testing.

| Tool | Purpose |
|---|---|
| Burp Suite | Intercepting, modifying and analysing HTTP requests |
| SQLmap | Automating SQL injection detection and exploitation |
| OWASP ZAP | Web application security testing |
| Nmap | Network discovery and database service identification |
| Burp Collaborator | Detecting out-of-band interactions |
| Hackvertor | Encoding and transforming request data |
| Browser Developer Tools | Inspecting application requests and responses |

### Burp Suite Features Relevant to SQLi

- **Proxy:** Intercepts HTTP requests.
- **Repeater:** Manually modifies and resends requests.
- **Intruder:** Automates controlled input variations.
- **Collaborator:** Detects external DNS and HTTP interactions.
- **Logger:** Records HTTP traffic.
- **Decoder:** Encodes and decodes different data representations.
- **Scanner:** Helps identify vulnerabilities through automated testing, where available.

---

# 10. Impact of SQL Injection

The impact depends on the database permissions, application architecture and vulnerability type.

Potential consequences include:

### Confidentiality
- Exposure of usernames and passwords.
- Disclosure of personal information.
- Access to confidential business records.
- Exposure of authentication tokens.

### Integrity
- Unauthorized modification of records.
- Manipulation of application data.
- Modification of user accounts.
- Alteration of transactions.

### Availability
- Deletion of important database records.
- Disruption of database operations.
- Resource exhaustion through expensive queries.

### Authentication and Authorization
- Authentication bypass.
- Unauthorized access to other accounts.
- Circumvention of application-level restrictions.

SQL injection can therefore affect all three components of the CIA Triad:

- Confidentiality
- Integrity
- Availability

---

# 11. SQL Injection Prevention and Remediation

The most effective defense against SQL injection is preventing untrusted input from changing the structure of SQL queries.

## 11.1 Prepared Statements and Parameterized Queries

This is the primary defense against SQL injection.

Instead of directly concatenating user input into SQL statements, use parameterized queries.

### Vulnerable Java Example

```java
String query = "SELECT * FROM users WHERE username = '" + username + "'";
Statement stmt = connection.createStatement();
ResultSet rs = stmt.executeQuery(query);
```

The user-controlled input becomes part of the SQL query itself.

### Secure Java Example

```java
String query = "SELECT * FROM users WHERE username = ?";

PreparedStatement stmt = connection.prepareStatement(query);

stmt.setString(1, username);

ResultSet rs = stmt.executeQuery();
```

The placeholder `?` separates SQL structure from user-supplied data.

Even if the input contains SQL syntax, it is treated as a parameter value rather than executable SQL.

## 11.2 Input Validation

Validate user input according to expected formats.

For example:

- Numeric fields should accept numeric values.
- Email fields should follow appropriate email validation.
- IDs should follow expected data types.
- Unnecessary input should be rejected.

Input validation is an additional security layer, not a replacement for parameterized queries.

## 11.3 Least Privilege

Database accounts should have only the permissions necessary for their intended functions.

For example:

- A read-only application account should not have unnecessary UPDATE or DELETE privileges.
- Application accounts should not have database administrator privileges.
- Separate database accounts should be used for different application functions where appropriate.

## 11.4 Avoid Dynamic SQL

Avoid constructing SQL queries through direct string concatenation.

Where dynamic SQL is necessary, use safe parameter binding and strict allowlists for SQL identifiers that cannot be parameterized, such as table or column names.

## 11.5 Secure Error Handling

Do not expose detailed database errors to end users.

Detailed error messages may reveal:

- Database technology.
- Table names.
- Column names.
- Query structure.
- Internal application information.

Log detailed errors securely on the server while returning generic error messages to users.

## 11.6 Defense in Depth

Additional security measures include:

- Properly configured WAFs.
- Secure database permissions.
- Regular code reviews.
- SAST and DAST testing.
- Dependency and framework updates.
- Security logging and monitoring.

A WAF should be considered an additional defensive layer, not the primary solution.

---

# 12. SQL Injection Detection and Monitoring

Organizations should monitor application and database activity for suspicious behavior.

Potential indicators include:

- Unexpected SQL syntax in application inputs.
- Repeated database errors.
- Unusual query patterns.
- Abnormal database response times.
- Unexpected access to sensitive tables.
- Unusual database account activity.
- Unexpected outbound DNS or HTTP requests from database servers.

Security teams can use:

- Web application logs.
- Database audit logs.
- SIEM platforms.
- EDR/XDR solutions.
- WAF logs.

Correlating application and database logs can help identify suspicious activity.

---

# 13. Common SQL Injection Misconceptions

| Misconception | Reality |
|---|---|
| SQL injection only happens in login forms. | It can occur in any input incorporated into SQL queries. |
| Only URL parameters are vulnerable. | POST bodies, cookies, headers and XML inputs can also be vulnerable. |
| A WAF completely prevents SQL injection. | WAFs can provide additional protection but cannot replace secure query construction. |
| SQL injection always produces database errors. | Blind SQL injection may not produce visible errors. |
| SQL injection only affects MySQL. | Multiple relational database systems can be vulnerable. |
| Input validation alone prevents SQL injection. | Parameterized queries are the primary defense. |
| Encoding input makes it safe. | Encoding does not fix unsafe SQL query construction. |

---

# 14. Key Concepts to Remember

| Concept | Important Point |
|---|---|
| SQL Injection | Manipulation of SQL queries through untrusted input |
| In-Band SQLi | Injection and retrieval through the same communication channel |
| Error-Based SQLi | Uses database errors to extract information |
| UNION-Based SQLi | Combines results from multiple SELECT queries |
| Blind SQLi | Infers database information through application behavior |
| Boolean-Based SQLi | Uses TRUE/FALSE conditions |
| Time-Based SQLi | Uses response delays |
| Out-of-Band SQLi | Uses a separate communication channel |
| Second-Order SQLi | Stored input becomes dangerous when reused in a later query |
| Database Enumeration | Discovering database structure and metadata |
| WAF Bypass | Evading poorly implemented request filtering |
| Parameterized Queries | Separates SQL instructions from user input |

---

# 15. Conclusion

SQL Injection remains one of the most important web application security vulnerabilities because it can expose sensitive database information and compromise application functionality.

Understanding SQL injection requires knowledge of SQL syntax, database architecture, HTTP communication, application behavior and secure coding practices.

Practical testing involves identifying injection points, analysing application responses, understanding database-specific behavior and determining the extent of the vulnerability.

From a defensive perspective, parameterized queries, least-privilege database permissions, secure error handling and continuous security testing are essential.

Learning SQL injection is not just about understanding payloads. It is about understanding how applications communicate with databases, how trust boundaries fail and how those weaknesses can be prevented.

## References

- PortSwigger Web Security Academy – SQL Injection
- OWASP Web Security Testing Guide – Testing for SQL Injection
- OWASP SQL Injection Prevention Cheat Sheet
- CWE-89 – Improper Neutralization of Special Elements used in an SQL Command
- PostgreSQL Documentation
- MySQL Documentation
- Microsoft SQL Server Documentation