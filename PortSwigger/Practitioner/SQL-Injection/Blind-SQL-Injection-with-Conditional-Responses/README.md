# SQL Injection: Blind SQL Injection with Conditional Responses

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind SQL Injection via Boolean-Based Conditional Response (CWE-89)

## Lab Objective
A tracking cookie (`TrackingId`) is used directly in a SQL query with no results or errors ever returned to the client — the only observable signal is whether a `Welcome back` message appears on the page. Exploit this single binary signal to extract the administrator's password, character by character, and log in.

## Vulnerability Overview
This lab represents the point where SQL injection techniques diverge sharply from everything covered earlier in this portfolio. `UNION`-based attacks depend entirely on the application reflecting query results somewhere in the response — here, that channel doesn't exist at all. The query's actual output (rows, columns, data) is never shown to the user in any form; the only thing that changes based on the query's result is a single cosmetic UI element.

That single bit of information — "did the `Welcome back` message appear, yes or no" — is still enough to exploit the injection fully, because it functions as a **boolean oracle**: the application effectively answers one true/false question per request. The core technique is to replace the straightforward value the query expects with a conditional SQL expression, structured so that the overall query only returns a matching row (and therefore triggers `Welcome back`) when the attacker's injected condition evaluates to true. By asking a long sequence of carefully chosen true/false questions — "is the password's first character greater than 'm'?", "does it equal 's'?", "is it exactly 20 characters long?" — the entire password can be reconstructed one bit (or one character) at a time, despite never seeing a single byte of actual data returned.

## Methodology
1. **Identify the injection point and the oracle.** Confirm the `TrackingId` cookie value is used in a SQL query, and identify the exact observable behavior (the `Welcome back` message) that changes based on the query's result.
2. **Establish a true/false baseline.** Inject an always-true condition and confirm the oracle fires; inject an always-false condition and confirm it doesn't. This proves the injection point is controllable and the oracle is reliable before investing effort in extraction.
3. **Confirm a specific row can be targeted.** Test a condition referencing the `users` table and the `administrator` username specifically, to ensure the oracle can be scoped to the exact target row rather than any row in the table.
4. **Determine the password's length** using a `LENGTH()`-based condition, incrementing or binary-searching until the oracle flips from true to false.
5. **Extract the password character by character**, using either a binary-search comparison (`>` against the midpoint of the character range) or direct equality checks, repeating the process independently for each character position.
6. **Automate the character-by-character extraction with Burp Intruder**, since doing this entirely by hand across potentially dozens of characters is impractical.
7. **Log in with the fully reconstructed password.**

---

## Exploitation

**Step 1:** Intercept any request carrying the `TrackingId` cookie in Burp Suite and send it to **Repeater**:

```http
GET / HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: TrackingId=u5YD3PapBcR4lN3e7Tj4; session=...
```

**Step 2:** Modify the cookie to inject an always-true condition:

```
Cookie: TrackingId=xyz' AND '1'='1
```

The response includes the `Welcome back` message — confirming the injected SQL is being evaluated and the condition's truth value directly controls this behavior.

**Step 3:** Change the condition to an always-false equivalent:

```
Cookie: TrackingId=xyz' AND '1'='2
```

The `Welcome back` message disappears — confirming the oracle responds reliably to the condition's truth value in both directions, which rules out false positives/negatives before proceeding further.

**Step 4:** Confirm the oracle can be scoped to a specific user and that a subquery returning a value works as expected:

```
Cookie: TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND 'a'='a')='a
```

The message appears, confirming subquery-based conditions against the `users` table function correctly within the injection.

**Step 5:** Determine the password length using `LENGTH()`:

```
Cookie: TrackingId=xyz' AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)=20)='a
```

Testing different length values, the message appears only when the condition is set to `20`, confirming the administrator's password is exactly 20 characters long.

**Step 6:** Extract each character using a binary-search comparison against the ASCII/alphanumeric midpoint, narrowing the range with each test. For the first character:

```
Cookie: TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')>'m
```

A true result narrows the search to the upper half of the alphanumeric range; a false result narrows it to the lower half. Repeating this bisection (typically 5–7 requests per character, given the hint that the password is lowercase alphanumeric only) pins down the exact character.

**Step 7:** Automate this process in **Burp Intruder**, using a payload position for the character value at each index and a **Cluster Bomb** or scripted sniper attack iterating through the alphanumeric character set for each of the 20 positions, filtering results by response length or the presence of the `Welcome back` string to identify the matching character per position.

**Step 8:** After reconstructing all 20 characters, the full administrator password is assembled. Log in using `administrator` and the extracted password to solve the lab.

**Why it works:** The injection point is identical in kind to every `UNION`-based lab in this series — unsanitised input reaching a SQL query — but the exploitation technique is forced to adapt entirely to the absence of any data channel. A single bit of application behavior, faithfully tied to the truth value of an injected condition, is sufficient to exfiltrate arbitrary data given enough individually-answered yes/no questions; the real cost isn't technical complexity but sheer request volume, which is exactly why this class of attack is impractical without tooling like Intruder, despite being conceptually no harder than the `UNION` labs to understand.

## Impact
- **Full data exfiltration with zero visible query output** — demonstrates that "the response doesn't show database results" is not a mitigating factor in any way; it only changes which technique is required, not whether exploitation is possible.
- **Significantly higher request volume than `UNION`-based extraction**, which is itself a detectable pattern — a real attacker running this technique against a production system would generate a distinctive, high-volume sequence of near-identical requests differing only in the injected condition, a strong candidate for WAF/anomaly detection signatures.
- **Generalizes to any single-bit application behavior**, not just a "Welcome back" message — subtly different response lengths, different HTTP status codes, different page layouts, or even timing differences (covered in later labs in this series) can all serve as an equally valid oracle once identified.

## Remediation
- **Use parameterised queries (prepared statements) universally.** As with every SQL injection lab in this portfolio, this remains the only fix that closes the injection point itself — blind extraction techniques are only possible because the query's *structure* can be altered by input in the first place; preventing that structural alteration defeats boolean-based, error-based, and time-based blind techniques simultaneously.
- **Rate-limit and monitor for abnormal request volumes against a single endpoint/parameter**, since blind SQL injection extraction inherently requires hundreds of highly similar requests — a pattern that's detectable at the network/WAF layer even without understanding the underlying vulnerability.
- **Avoid tying any observable behavior directly to the result of a database query built from unsanitised input** — even cosmetic UI differences become a viable oracle, so this consideration belongs in threat modeling, not just backend security review.
- **Apply least-privilege database accounts and generic error handling**, as defense-in-depth measures that don't fix this specific technique but limit the blast radius of successful exploitation.

## Key Takeaways
- Blind SQL injection is not a weaker or lesser vulnerability than visible/`UNION`-based injection — it has identical data-exfiltration potential, with the only real difference being exploitation cost (number of requests required) rather than exploitation ceiling.
- Binary search dramatically reduces the number of requests needed per character compared to naive sequential testing (checking `='a'`, then `='b'`, then `='c'`...) — recognizing when a comparison operator (`>`, `<`) can be used instead of equality is what makes this technique practical at all.
- Establishing a reliable true/false baseline (`'1'='1'` vs `'1'='2'`) before attempting any actual data extraction is a non-negotiable first step — without it, a failed extraction attempt could mean either "the character is wrong" or "the oracle itself isn't working," and there's no way to distinguish the two without that baseline.
- Burp Intruder's Cluster Bomb/Sniper modes aren't optional conveniences for this technique — manually testing 20 characters × up to 7 comparisons each by hand is exactly the kind of repetitive, mechanical task that automation exists for, and recognizing that early saves significant time.

## Tools Used
- Burp Suite (Proxy, Repeater, Intruder)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.