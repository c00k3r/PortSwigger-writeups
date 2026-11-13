# Detecting NoSQL Injection

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
NoSQL Injection – MongoDB – Syntax Injection

## Lab Objective
Identify and exploit a NoSQL injection vulnerability in the product category filter to make the application display unreleased products.

## Vulnerability Overview

NoSQL injection is a vulnerability that occurs when an application incorporates untrusted user input into NoSQL database queries without properly validating or handling that input.

Unlike traditional SQL injection, which targets relational databases and SQL query syntax, NoSQL injection targets databases such as MongoDB that use different query structures and operators.

MongoDB supports document-oriented queries, JavaScript expressions in certain contexts, and operators such as `$ne`, `$gt`, and `$regex`.

If user input is incorporated unsafely into these queries, an attacker may manipulate the database query's logic.

## Methodology

1. Opened the application and inspected its product category filtering functionality.
2. Observed that the category filter was powered by a MongoDB NoSQL database.
3. Intercepted the category filtering request using Burp Suite.
4. Sent the request to Burp Repeater for controlled testing.
5. Tested the input using special characters to identify possible NoSQL syntax injection.
6. Observed differences in application responses when modifying the input.
7. Used JavaScript syntax manipulation to alter the original database query's logic.
8. Modified the query condition so that the application returned products beyond the originally selected category.
9. Verified that unreleased products became visible and the lab was solved.

## Exploitation Concepts

### 1. Identifying the Injection Point

The application uses a category filter to retrieve products from MongoDB.

A typical application query might conceptually resemble:

```javascript
db.products.find({
    category: userInput
});
```

If the application constructs queries unsafely, user-controlled input may influence the query structure.

### 2. Testing for NoSQL Injection

Special characters can help identify whether user input is interpreted as part of a NoSQL query or expression.

Examples include:

```text
'
"
\
{
}
```

Unexpected errors, changes in response content, or differences in application behavior can indicate possible injection.

However, an error alone does not prove that a NoSQL injection vulnerability exists. Further testing is required.

### 3. Understanding Syntax Injection

Some NoSQL injection vulnerabilities occur when user input is incorporated into JavaScript expressions used by the database.

For example, an application might construct a condition conceptually similar to:

```javascript
this.category == 'Gifts'
```

If user input is directly inserted into this expression, an attacker may be able to manipulate its logical conditions.

This is known as NoSQL syntax injection.

### 4. Manipulating Query Logic

The objective is to alter the original category condition so that it no longer restricts results to the selected category.

By manipulating the expression's logic, the application may return additional products, including products that were not originally displayed.

### 5. Verifying the Vulnerability

After modifying the category filter, inspected the returned product listings.

The appearance of previously unavailable or unreleased products confirmed that the original filtering logic had been bypassed.

The lab was successfully solved.

## NoSQL Injection vs SQL Injection

| Feature | SQL Injection | NoSQL Injection |
|---|---|---|
| Database type | Relational databases | Non-relational databases |
| Query language | SQL | Database-specific query structures |
| Common targets | MySQL, PostgreSQL, SQL Server | MongoDB, CouchDB |
| Query manipulation | SQL syntax and operators | Query objects, operators and expressions |
| Common operators | OR, AND, UNION | $ne, $gt, $regex |
| Data structure | Tables and rows | Documents and collections |

## Common NoSQL Injection Operators

MongoDB provides several query operators that may become dangerous when user-controlled input is incorporated into database queries unsafely.

| Operator | Purpose |
|---|---|
| `$ne` | Matches values not equal to a specified value |
| `$gt` | Matches values greater than a specified value |
| `$lt` | Matches values less than a specified value |
| `$gte` | Matches values greater than or equal to a specified value |
| `$lte` | Matches values less than or equal to a specified value |
| `$in` | Matches any value in a specified array |
| `$nin` | Matches values not present in a specified array |
| `$regex` | Matches values against a regular expression |

These operators are legitimate database functionality. They become security concerns when applications allow attackers to control query structures or operators.

## Root Cause

The vulnerability occurs when the application incorporates user-controlled input into MongoDB queries or JavaScript expressions without proper validation and safe query construction.

This allows an attacker to manipulate the intended query logic and retrieve records that should remain inaccessible through the normal application functionality.

## Remediation

- Use structured MongoDB query objects instead of constructing query expressions through string concatenation.
- Validate user input against expected data types and formats.
- Prevent users from supplying arbitrary query operators.
- Avoid evaluating untrusted input as JavaScript.
- Apply allowlists for permitted fields and query operators.
- Implement appropriate authorization checks on the server side.
- Follow the principle of least privilege for database accounts.
- Avoid exposing detailed database errors to users.

## Key Takeaways

- NoSQL injection targets non-relational databases and their query mechanisms.
- MongoDB uses documents and collections rather than traditional relational tables.
- Special characters can help identify potential syntax injection vulnerabilities.
- JavaScript syntax injection can allow attackers to manipulate query conditions.
- MongoDB operators such as `$ne`, `$gt`, and `$regex` can be abused when applications allow uncontrolled query construction.
- NoSQL injection can expose records that are hidden or restricted by application logic.
- Secure query construction and strict input validation are essential defenses.

## References

- PortSwigger Web Security Academy – NoSQL Injection
- OWASP Web Security Testing Guide
- OWASP NoSQL Security Cheat Sheet
- MongoDB Documentation – Query Documents
- CWE-943 – Improper Neutralization of Special Elements in Data Query Logic