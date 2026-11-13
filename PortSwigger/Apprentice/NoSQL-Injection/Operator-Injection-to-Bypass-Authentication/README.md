# Exploiting NoSQL Operator Injection to Bypass Authentication

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
NoSQL Injection – MongoDB – Operator Injection – Authentication Bypass

## Lab Objective

Exploit a NoSQL injection vulnerability in the application's login functionality to bypass authentication and log in as the `administrator` user.

The lab provides the following legitimate account credentials:

- Username: `wiener`
- Password: `peter`

## Vulnerability Overview

NoSQL operator injection occurs when an application allows user-controlled input to manipulate database query operators.

MongoDB uses operators such as `$ne`, `$gt`, `$lt`, and `$regex` to perform database operations.

When an application constructs database queries unsafely, an attacker may inject these operators to modify the intended query logic.

Unlike traditional SQL injection, which manipulates SQL statements, NoSQL operator injection targets the structure of NoSQL query objects.

In this lab, the login functionality uses MongoDB and is vulnerable to operator injection, allowing authentication checks to be manipulated.

## Methodology

1. Opened the application and inspected its login functionality.
2. Logged in using the provided credentials (`wiener:peter`) to understand the normal authentication process.
3. Intercepted the login request using Burp Suite Proxy.
4. Sent the request to Burp Repeater for controlled testing.
5. Examined how the username and password parameters were submitted.
6. Identified that the application processed login credentials through a MongoDB query.
7. Tested whether the application accepted manipulated input instead of ordinary string values.
8. Used MongoDB query operators to modify the authentication condition.
9. Manipulated the login query to authenticate as the administrator without knowing the administrator's password.
10. Successfully accessed the administrator account and solved the lab.

## Exploitation Concepts

### 1. Understanding the Login Query

A typical MongoDB authentication query might look like:

```javascript
db.users.findOne({
    username: username,
    password: password
});
```

The application searches for a user whose username and password match the supplied values.

If the application allows attackers to control the structure of these query parameters, the authentication logic may be manipulated.

### 2. Understanding MongoDB Operators

MongoDB provides operators that control how database conditions are evaluated.

Some important operators include:

| Operator | Description |
|---|---|
| `$ne` | Matches values not equal to a specified value |
| `$gt` | Matches values greater than a specified value |
| `$lt` | Matches values less than a specified value |
| `$gte` | Matches values greater than or equal to a specified value |
| `$lte` | Matches values less than or equal to a specified value |
| `$in` | Matches any value in a specified array |
| `$nin` | Matches values not present in a specified array |
| `$regex` | Performs pattern matching |

### 3. Understanding the Injection

Consider the following MongoDB query:

```javascript
db.users.findOne({
    username: "administrator",
    password: "somepassword"
});
```

The application expects both conditions to match.

However, if an attacker can inject a MongoDB operator such as `$ne`, the password condition can potentially be manipulated.

For example, the following condition:

```javascript
password: {
    "$ne": null
}
```

Matches password values that are not equal to null.

If the application accepts this object as a query condition, it no longer performs an exact password comparison.

### 4. Bypassing Authentication

The objective was to manipulate the login request so that MongoDB evaluated the password condition differently from the intended exact-match comparison.

By exploiting the application's unsafe handling of query operators, authentication could be bypassed without knowing the administrator's actual password.

The application then authenticated the request as the administrator.

### 5. Successful Authentication

After manipulating the login request, accessed the administrator account.

The successful login confirmed that the application's authentication mechanism was vulnerable to NoSQL operator injection.

## Example of Vulnerable Implementation

A vulnerable application might directly incorporate request parameters into a MongoDB query:

```javascript
const user = await db.collection("users").findOne({
    username: req.body.username,
    password: req.body.password
});
```

If the application accepts objects instead of enforcing string values, an attacker may be able to supply MongoDB operators as input.

## Root Cause

The root cause was the application's failure to properly validate the structure and data types of user-supplied login parameters.

Instead of ensuring that username and password values were ordinary strings, the application allowed query operators to influence database query evaluation.

This enabled manipulation of the authentication condition.

## Remediation

- Enforce strict data-type validation for username and password parameters.
- Accept only strings for authentication fields.
- Reject unexpected objects, arrays and nested JSON structures.
- Never allow users to supply arbitrary MongoDB query operators.
- Use secure authentication libraries and properly implemented credential verification.
- Store passwords using strong password hashing algorithms such as Argon2id or bcrypt.
- Implement rate limiting and appropriate authentication monitoring.
- Apply least-privilege permissions to database accounts.

## Key Takeaways

- NoSQL injection can occur when applications allow users to manipulate database query structures.
- MongoDB operators such as `$ne`, `$gt`, and `$regex` can be abused when user input is not properly validated.
- NoSQL operator injection can lead to authentication bypass.
- JSON input does not automatically make an application secure.
- Strict data-type validation is important when handling authentication parameters.
- Applications should construct database queries safely and never trust user-controlled query objects.

## References

- PortSwigger Web Security Academy – NoSQL Injection
- OWASP NoSQL Security Cheat Sheet
- MongoDB Documentation – Query and Projection Operators
- CWE-943 – Improper Neutralization of Special Elements in Data Query Logic