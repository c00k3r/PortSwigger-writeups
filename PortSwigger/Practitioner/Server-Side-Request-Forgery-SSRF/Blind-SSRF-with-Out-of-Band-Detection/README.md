# SSRF: Blind SSRF with Out-of-Band Detection

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind Server-Side Request Forgery (CWE-918)

## Lab Objective
The site's analytics software fetches whatever URL is submitted in the `Referer` header when a product page loads. Confirm this blind SSRF by causing an HTTP request to a Burp Collaborator server and observing the resulting out-of-band interaction.

## Vulnerability Overview
Every previous SSRF lab in this series relied on the application reflecting some observable result — a stock count, an admin panel's HTML, a rendered response. This lab removes that feedback entirely: the vulnerable request originates not from a parameter the user directly submits for processing, but from a **header the analytics component passively consumes and fetches in the background**, with the result never surfacing anywhere in the HTTP response the attacker sees.

The `Referer` header is normally supplied by browsers to indicate which page a request originated from — it's informational, not something applications typically treat as untrusted input requiring validation. That's precisely the gap here: the analytics software appears to take the header's value at face value and makes a server-side request to it, likely to log or verify the referring source, without considering that an attacker fully controls this header's contents on any request they send directly (as opposed to a browser-generated one).

Because there's no reflected data and no application behavior to directly observe, this is a textbook case for **out-of-band (OOB) confirmation** via Burp Collaborator — proving the vulnerability exists by making the server interact with infrastructure the attacker controls and monitors independently of the original request/response cycle.

## Methodology
1. **Identify the passive injection point.** Recognize that the `Referer` header, while not a conventional user-input field, is fully attacker-controlled on any request sent outside of normal browser navigation (i.e. via Burp Repeater).
2. **Confirm the feature's trigger condition.** The lab description specifies the analytics fetch happens on product page load — any request to a product page is a candidate for testing.
3. **Generate a Burp Collaborator payload** to use as the injected `Referer` value.
4. **Submit the crafted request** and allow time for the server-side analytics process (which may run asynchronously, after the response has already been returned) to complete its fetch.
5. **Poll Collaborator** for the resulting interaction, confirming the server made the outbound request.

---

## Exploitation

**Step 1:** Browse to any product page and intercept the request in Burp Suite:

```http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Referer: https://LAB-ID.web-security-academy.net/
```

**Step 2:** Open **Burp > Collaborator client** and generate a unique payload, e.g.:

```
a1b2c3d4e5f6g7h8i9j0.oastify.com
```

**Step 3:** Send the request to **Repeater**. Replace the `Referer` header value with the generated Collaborator domain:

```http
GET /product?productId=1 HTTP/1.1
Host: LAB-ID.web-security-academy.net
Referer: http://a1b2c3d4e5f6g7h8i9j0.oastify.com
```

**Step 4:** Send the request. The application returns its normal product page response — there is no indication in the HTTP response itself that anything unusual happened, consistent with this being a blind vulnerability.

**Step 5:** Wait briefly (the analytics fetch may be processed asynchronously, outside the main request/response cycle, so the interaction may not be immediate), then return to the **Collaborator client** and click **Poll now**.

**Step 6:** The interaction log shows a DNS lookup and/or HTTP request originating from the lab's server, targeting the Collaborator payload — confirming the analytics software fetched the attacker-supplied `Referer` value exactly as hypothesized.

**Step 7:** This confirmed out-of-band interaction is sufficient to solve the lab.

**Why it works:** The vulnerability exists because the analytics component treats the `Referer` header as a trustworthy, purely informational value rather than untrusted input — a reasonable assumption for organically browser-generated traffic, but one that collapses the moment a request is crafted directly rather than navigated to normally. Since the resulting fetch happens entirely server-side with no part of its outcome returned to the client, there is no in-band way to confirm the vulnerability; the Collaborator interaction itself — proof that the server reached out to attacker-controlled infrastructure — is the entire exploit and the entire proof.

## Impact
- **Confirms SSRF via an unconventional, easily overlooked injection point.** Headers like `Referer`, `X-Forwarded-For`, and `User-Agent` are frequently logged, parsed, or acted upon server-side without the same scrutiny applied to query parameters or form fields — this lab is a reminder to test headers as first-class attack surface, not just request bodies.
- **Demonstrates asynchronous/background processing as an SSRF trigger**, distinct from a feature that fetches a URL as part of generating its immediate response — the fetch here happens as a side effect of analytics logging, which may not even block the original response at all.
- **Confirmed blind SSRF is a foundation for further escalation** — in a real engagement, the same technique would be the first step toward probing internal network services, cloud metadata endpoints, or chaining with an internal-only vulnerability, exactly as demonstrated with reflected SSRF earlier in this series.

## Remediation
- **Never treat client-supplied headers as trusted input**, including conventionally "informational" ones like `Referer` — any header value that influences server-side behavior, especially triggering an outbound request, must be validated with the same rigor as any other user input.
- **Avoid making server-side requests based on client-controlled values for analytics or logging purposes entirely** where possible — log the raw header value for later analysis instead of actively fetching it.
- **If fetching referrer or similar URLs is a genuine business requirement**, apply strict allowlisting of permitted domains/schemes, and resolve and validate the destination IP before connecting, exactly as recommended for every other SSRF vector in this portfolio.
- **Apply network-level egress controls and monitoring**, since blind SSRF triggered via background processes can be harder to detect through conventional response-based security testing alone — outbound traffic monitoring becomes a necessary complementary control.
- **Audit all request headers for server-side consumption**, not just obviously "special" ones — any header a backend reads and acts upon is effectively a user-input field, regardless of its conventional purpose.

## Key Takeaways
- Blind vulnerabilities aren't limited to obviously "quiet" parameters — a header as mundane as `Referer`, consumed by a background analytics process, is just as valid an SSRF vector as a prominent `stockApi` parameter.
- Confirmed OOB interaction is sufficient proof of exploitability even with zero in-band feedback — the absence of a visible effect in the response is not evidence of safety, only evidence that a different detection method is required.
- Asynchronous, background-processed side effects (analytics, logging, webhooks) are a distinct and often under-tested category of SSRF trigger compared to synchronous, response-generating fetches — worth deliberately testing for in any application with analytics, telemetry, or webhook-style integrations.
- This lab closes out the "Circumventing SSRF defenses" learning path by returning to the simplest possible exploitation mechanic (no filter to bypass at all) while demonstrating the hardest possible detection scenario (fully blind) — a useful reminder that defense complexity and exploitation difficulty don't always move together.

## Tools Used
- Burp Suite (Proxy, Repeater, Collaborator client)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.