# API Testing: Finding and Exploiting an Unused API Endpoint

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Improper Inventory Management / Broken Function Level Authorization (OWASP API9:2023, API5:2023)

## Lab Objective
Exploit a hidden API endpoint — one still deployed but never called by the application's frontend and not listed in the documentation — to purchase a product ("Lightweight l33t Leather Jacket") without actually paying for it.

## Vulnerability Overview
APIs evolve over time: endpoints get superseded by newer versions, feature flags get toggled off in the UI, or functionality is split across multiple HTTP methods on the same path during development. The problem is that **removing a reference to an endpoint is not the same as removing the endpoint**. If the backend route is still live, it remains fully reachable by anyone who can guess or derive its path — regardless of whether any frontend code, documentation, or navigation link points to it.

This lab specifically targets a case where an endpoint's intended, safe behaviour (e.g. viewing a product) is exposed via one HTTP method, while a *different* method on the same path exposes unintended, more privileged behaviour (e.g. purchasing without proper checks) — and that second method was never meant to be reachable by normal users, only discoverable by actively probing the API rather than trusting documentation or UI flow.

**Why this differs from the "API documentation" lab:** that lab's vulnerable endpoint was disclosed by leaked docs. Here, there is no documentation to read — the endpoint must be *derived* by reasoning about REST conventions and reacting to the server's own error messages, which is a core black-box API testing skill.

## Methodology
1. **Map the known API surface.** Proxy the checkout flow through Burp Suite and record every endpoint the frontend actually calls — product listing, cart, checkout.
2. **Reason about REST conventions.** A RESTful API conventionally exposes the same resource path under multiple HTTP methods (`GET` to read, `POST` to create, `PUT`/`PATCH` to update, `DELETE` to remove). If `GET /api/products/1/price` exists, a `PATCH` or `POST` on that same path is a reasonable hypothesis worth testing, even with nothing pointing to it.
3. **Send a known-good request to Repeater** and begin methodically changing the HTTP method, observing how the server's error messages change — a verbose or oddly-specific error (e.g. "missing parameter X") is a strong signal that the endpoint exists and is being processed, not just rejected outright.
4. **Iterate using the server's own responses as a guide.** Each error message narrows down the correct method, required parameters, and expected format — essentially letting the backend document itself through trial and error.
5. **Construct a valid request** once the required method and parameters are known, and trigger the unintended behaviour.

---

## Exploitation

**Step 1:** Log in as `wiener:peter` and browse to the target product, noting the normal purchase flow goes through the cart and a standard checkout endpoint such as:

```http
POST /api/checkout HTTP/1.1
```

**Step 2:** While exploring the product page's requests in Burp, identify a product-specific endpoint not tied to checkout, e.g.:

```http
GET /api/products/1/price HTTP/1.1
```

This returns the product's price — read-only, and seemingly harmless.

**Step 3:** Send this request to Repeater and change the HTTP method from `GET` to `PATCH`, with no body:

```http
PATCH /api/products/1/price HTTP/1.1
```

**Step 4:** The server responds with an error indicating a required field is missing — for example, something to the effect of expecting a JSON body with a price value. This confirms `PATCH` is accepted on this path and is processed server-side, not just rejected as an invalid method.

**Step 5:** Construct a request body matching what the error implies, setting the price to a negligible value:

```http
PATCH /api/products/1/price HTTP/1.1
Content-Type: application/json

{
  "price": 0
}
```

**Step 6:** The server accepts the update and confirms the change. The product's price is now effectively free.

**Step 7:** Return to the storefront, add the now-zero-priced jacket to the cart, and complete checkout through the normal flow. The purchase succeeds, and the lab is marked as solved.

**Why it works:** The `PATCH` method on `/api/products/1/price` was almost certainly a legitimate internal/admin capability for updating product pricing — but the server performed no authorization check confirming the caller had admin privileges, and no frontend link ever exposed it to standard users. "Not linked" was functioning as the only protection, which collapses the moment an attacker reasons about REST semantics and probes adjacent methods on a path they already know exists.

## Impact
- **Business logic bypass:** an attacker can arbitrarily modify product pricing, resulting in direct financial loss.
- **Broken function-level authorization:** administrative capability is reachable by an unprivileged, authenticated user — and in many real-world cases, even unauthenticated callers.
- **Generalizable risk:** this pattern applies to any REST resource where read and write operations share a path — stock levels, account roles, order statuses — all become candidates for the same method-switching technique.
- **Shadow/zombie API risk:** deprecated or internal-only endpoints that remain deployed are a leading real-world cause of API breaches, precisely because they fall outside normal security review once "removed" from the visible product.

## Remediation
- **Enforce authorization independently of discoverability.** Every method on every endpoint must verify the caller's role and permissions server-side — never assume an endpoint is safe because it isn't linked or documented.
- **Decommission unused endpoints completely**, not just their frontend references. If functionality is deprecated, the route should be removed or explicitly disabled, not left live and reachable.
- **Maintain an accurate API inventory.** Teams should track every deployed endpoint and method centrally (an API gateway or registry), rather than relying on frontend code as the source of truth for what "exists."
- **Avoid verbose error messages for unauthenticated/under-authorized requests.** Validation errors that reveal expected parameter names and formats make this kind of enumeration significantly faster for an attacker.
- **Apply the same input validation and business-logic checks regardless of HTTP method** — a price update should always be bounds-checked and privilege-gated, whether triggered via `PATCH`, `POST`, or any other verb.

## Key Takeaways
- REST convention itself is a reconnaissance tool: if one method exists on a path, systematically test the others.
- A server's own error messages, when verbose, effectively self-document an otherwise undocumented endpoint — treat every error as information, not just a dead end.
- "No frontend link" and "no documentation" are not security controls; they only raise the cost of discovery, not the cost of exploitation once discovered.
- This is a function-level authorization failure, distinct from the object-level failure in the earlier API documentation lab — same root cause (no server-side authorization), different mechanism (hidden method vs. hidden endpoint).

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.