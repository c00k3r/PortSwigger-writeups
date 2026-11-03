# API Testing: Exploiting Server-Side Parameter Pollution in a Query String

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Server-Side Parameter Pollution / SSPP (OWASP API8:2023 — Security Misconfiguration)

## Lab Objective
Log in as the `administrator` and delete `carlos`, by exploiting a discrepancy between how the front end and an internal back-end API parse the same query string during a password reset flow.

## Vulnerability Overview
Server-side parameter pollution happens when a client-facing application takes user input and uses it to construct a brand-new query string for an internal, server-to-server API call — without properly encoding or isolating that input first. If the user-supplied value itself contains query-string metacharacters (`&`, `#`, `=`), and those characters aren't encoded before being concatenated into the internal request, the attacker effectively gets to inject their own additional parameters into a request they were never supposed to control.

This is conceptually similar to SQL injection, but the "query" being broken out of is a URL query string instead of SQL syntax, and the target is an *internal* API the attacker can't reach directly — they're smuggling parameters into it through a trusted intermediary. Three characters make this possible:

| Character | Encoded | Effect in an internal query string |
|-----------|---------|-------------------------------------|
| `&` | `%26` | Starts a new parameter-value pair |
| `=` | `%3D` | Assigns a value to an injected parameter |
| `#` | `%23` | Truncates everything after it (fragment delimiter) |

Because the application performs the concatenation server-side, before forwarding to the internal API, these injected characters never touch the browser's own URL parsing — they're invisible to the user and to any front-end validation, and only take effect once the back-end builds and sends its own request.

## Methodology
This lab has no single obvious entry point — it requires systematically reverse-engineering an undocumented internal API purely from how its error messages change in response to crafted input. The overall approach:
1. **Identify a feature that triggers a server-side request built from user input.** A password reset flow is a strong candidate, since it typically requires the server to look up account details using the submitted identifier.
2. **Establish a response baseline**, then deliberately break input in small, controlled ways and read every error message closely — each one leaks a detail about how the internal API is structured.
3. **Probe for parameter injection** using encoded `&`, then confirm truncation is possible using encoded `#`.
4. **Enumerate hidden internal parameters** systematically once injection is confirmed, using Burp Intruder with a relevant wordlist.
5. **Inspect client-side JavaScript** for any direct references to internal API parameter names that might shortcut the brute-force step.
6. **Chain the discoveries** into a single crafted request that leaks data intended only for the legitimate internal caller.

---

## Exploitation

**Step 1 — Trigger the flow and capture the baseline request.**
With Burp's browser, trigger a password reset for the `administrator` user and locate the resulting request in **Proxy > HTTP history**:

```http
POST /forgot-password HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=...&username=administrator
```

Alongside it, a referenced script, `/static/js/forgotPassword.js`, is worth inspecting later — client-side JS often hints at backend parameter names.

**Step 2 — Confirm the request is consistent, then break it intentionally.**
Send the request to **Repeater** and resend it unmodified to confirm a stable baseline response. Then change `username` to a clearly invalid value, e.g. `administratorx`, and observe an `Invalid username` error — confirming the server validates this field against something, which means the value is actually being used in a lookup rather than silently ignored.

**Step 3 — Test for injection using an encoded `&`.**
Append a second parameter using a URL-encoded ampersand so it survives transmission without being parsed as a literal `&` by the outer request:

```
username=administrator%26x=y
```

The response changes to a `Parameter is not supported` error. This is the critical signal: the internal API parsed `x=y` as its own distinct parameter rather than as part of the `username` value — proof the front end is concatenating raw input into a second, internal query string.

**Step 4 — Test for truncation using an encoded `#`.**
Append an encoded fragment delimiter instead:

```
username=administrator%23
```

The response changes again, this time to a `Field not specified` error. Since `#` truncates everything after it in a URL, and removing the rest of the query caused a *different* required parameter to go missing, this implies the internal query normally includes an additional parameter — named `field` — positioned after `username`, which the application supplies automatically and which was just deleted by the truncation.

**Step 5 — Confirm the `field` parameter can be injected directly.**
Supply a `field` value explicitly, then truncate anything after it so no conflicting value is appended by the application itself:

```
username=administrator%26field=x%23
```

The response becomes `Invalid field`, confirming the server recognises `field` as a real, expected parameter and is now validating the attacker-supplied value against a list of legitimate options.

**Step 6 — Brute-force the valid values of `field`.**
Send the request to **Intruder**, mark the `field` value as the payload position:

```
username=administrator%26field=§x§%23
```

Use Burp's built-in **Server-side variable names** payload list and run the attack. Reviewing results by response code/length reveals that both `username` and `email` return a `200` response — distinct from the `Invalid field` error — meaning both are recognised, valid field names.

**Step 7 — Pivot from a known field to the actual target field.**
Confirm `email` directly:

```
username=administrator%26field=email%23
```

This returns the original baseline-style response, confirming `email` is accepted. Then revisit the earlier-noted `forgotPassword.js` file in **Proxy > HTTP history** — it references the password reset endpoint using a `reset_token` parameter. This is the real target: if `field` can be set to `email`, it can likely be set to `reset_token` as well, exfiltrating the actual secret needed to complete a reset.

**Step 8 — Exfiltrate the reset token.**
Change `field` accordingly:

```
username=administrator%26field=reset_token%23
```

The response now returns a valid password reset token for the `administrator` account — a value that was never meant to be directly retrievable, let alone for an account the attacker doesn't control.

**Step 9 — Complete the password reset.**
Navigate to the reset endpoint with the captured token:

```
/forgot-password?reset_token=<captured_token>
```

Set a new password for the `administrator` account.

**Step 10 — Log in and finish the objective.**
Authenticate as `administrator` using the new password, open the admin panel, and delete `carlos` to solve the lab.

**Why it works:** The application's `/forgot-password` handler builds a second, internal API request server-side, inserting the client-supplied `username` value directly into that request's query string without encoding the metacharacters it contains. The front end and the internal API effectively use two different trust boundaries for the same string — the front end treats `username` as one opaque field, but the internal API parses it as raw query syntax containing multiple parameters. That mismatch is the entire vulnerability: every subsequent step (confirming injection, finding `field`, enumerating its valid values, and finally requesting `reset_token`) is just methodically exploring what the internal API exposes once that boundary is broken.

## Impact
- **Full authentication bypass:** an attacker can obtain a password reset token for *any* account, including the administrator, without ever controlling that account's email.
- **Internal API exposure:** an API never meant to be reachable — let alone parameterizable — by end users becomes a fully interactive interface once its parameter names are known.
- **Systemic risk from error verbosity:** at no point was any credential or secret guessed; every step was derived purely from how error messages changed, underscoring how much information granular, distinct error responses leak to a patient attacker.
- **Privilege escalation to full admin control**, as demonstrated by the account deletion at the end of the chain.

## Remediation
- **Never concatenate raw user input into a second, server-built query string.** Treat this exactly like SQL injection: use structured request-building (e.g. a proper HTTP client library's parameter API) rather than string concatenation, so metacharacters are encoded automatically.
- **URL-encode all dynamic values** inserted into any internal request, not just ones ultimately rendered to the browser — encoding discipline shouldn't stop at the server boundary.
- **Apply strict allowlisting on internal parameters** (`field`, in this case) rather than accepting any value and only failing on recognisably invalid ones — the enumerability of `field`'s valid values was itself part of the attack surface.
- **Return generic, uniform error messages** for invalid input in security-sensitive flows. Each distinct error here (`Invalid username`, `Parameter is not supported`, `Field not specified`, `Invalid field`) functioned as an oracle, and collapsing them into one generic response would have broken the entire discovery chain.
- **Never expose a "field selector" pattern on a secrets-adjacent endpoint.** Allowing the client to dictate *which* server-side field is validated or returned (even indirectly) is inherently risky; the reset token lookup should only ever operate on a single, fixed, non-configurable field.
- **Rate-limit and monitor password-reset endpoints** for anomalous parameter patterns, since brute-forcing a parameter name (as done here via Intruder) produces a distinctive request pattern that's detectable.

## Key Takeaways
- Server-side parameter pollution is often invisible from the browser entirely — the injected characters only matter to a second request the client never sees, which makes it easy to miss without deliberately probing for it.
- Error messages are not noise — in a black-box API test, each distinct error is a data point, and systematically varying one input while reading the resulting error is often the only way to reverse-engineer an undocumented internal contract.
- This lab is a strong illustration of why trust boundaries must be re-validated at every hop: the front end correctly treated `username` as a safe string, but that safety assumption silently broke the moment it was reused to build a second request.
- Client-side JavaScript is a legitimate recon source even in "pure" server-side vulnerabilities — the `reset_token` parameter name here came directly from a front-end script, not from brute-forcing.

## Tools Used
- Burp Suite (Proxy, Repeater, Intruder)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.