# API Testing: Exploiting an API Endpoint Using Documentation

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
Broken Object Level Authorization / Exposed API Documentation (OWASP API1:2023, API9:2023)

## Lab Objective
Locate exposed API documentation that was never meant to be discoverable from the main application, and use it to identify an undocumented-to-the-UI administrative endpoint capable of deleting the user `carlos`.

## Vulnerability Overview
Modern applications frequently expose a REST or GraphQL API as a thin backend, with the frontend only calling a subset of the available endpoints. Developers commonly host machine-readable API documentation (Swagger/OpenAPI specs, or a human-readable docs page) to support internal development, integration, or testing — and just as commonly forget that this documentation is reachable by anyone who finds the URL, not just authorized developers.

This is a textbook **excessive data exposure via documentation** issue: the frontend intentionally never calls certain endpoints, so they appear "hidden" from normal use of the application. But hiding an endpoint from the UI is not the same as restricting access to it server-side. If the API itself enforces no additional authorization check on sensitive operations, discovering the endpoint through documentation is enough to call it directly and bypass whatever workflow the UI was meant to enforce.

**Why documentation matters to an attacker:** API docs typically disclose exact endpoint paths, accepted HTTP methods, required parameters, and expected request/response formats — effectively handing an attacker a map of the application's full attack surface, including functionality (like user deletion) that was never intended to be reachable by a standard user.

## Methodology
1. **Reconnaissance for hidden documentation.** Before assuming an API has no docs, check conventional discovery locations:
   - `/robots.txt` — frequently discloses disallowed paths that are not linked anywhere in the UI, often including the API root or docs path.
   - Common documentation paths: `/api`, `/api/docs`, `/swagger`, `/swagger-ui`, `/openapi.json`, `/api-docs`.
2. **Baseline.** Browse the application normally with Burp Suite proxying traffic, noting which API calls the frontend actually makes (via **Proxy > HTTP history**) — this establishes what's "intended" versus what documentation might later reveal as additional, unused functionality.
3. **Fetch and read the documentation.** Once found, the docs page enumerates every endpoint the backend supports, including ones the frontend never calls.
4. **Identify high-value endpoints.** Look specifically for endpoints touching user management, especially any supporting `DELETE`, `PUT`, or `PATCH` methods on a `users` resource.
5. **Call the endpoint directly** with Burp Repeater, bypassing the UI entirely.

---

## Exploitation

**Step 1:** Request `/robots.txt`:

```http
GET /robots.txt HTTP/1.1
Host: LAB-ID.web-security-academy.net
```

The response discloses a disallowed path pointing at the API root, something not linked anywhere in the visible site navigation:

```
User-agent: *
Disallow: /api/
```

**Step 2:** Browse to the disclosed path. It hosts a documentation page (or redirects to one) listing the API's available endpoints, methods, and parameters — effectively a Swagger/OpenAPI-style reference.

**Step 3:** Review the documented endpoints. Among the expected, UI-backed endpoints (e.g. fetching product or user info), the documentation discloses an endpoint intended for internal/administrative use:

```
DELETE /api/internal/v1/users/{username}
```

This endpoint is never called by the frontend — there is no "delete user" button anywhere in the application — but the API itself still accepts and processes the request from any caller, since no additional authorization check was implemented server-side.

**Step 4:** Log in with the provided low-privilege credentials (`wiener:peter`) to obtain a valid session, then send a direct request to the documented endpoint via Burp Repeater:

```http
DELETE /api/internal/v1/users/carlos HTTP/1.1
Host: LAB-ID.web-security-academy.net
Cookie: session=<wiener's session cookie>
```

**Step 5:** The server responds with a success status, confirming the user `carlos` has been deleted, and the lab is marked as solved.

**Why it works:** The vulnerability isn't in the endpoint's *logic* — deleting a user by username is a reasonable administrative operation to expose somewhere. The failure is twofold: the endpoint was discoverable purely through leaked documentation rather than genuinely restricted, and the backend performed no authorization check to confirm the caller (`wiener`, a standard user) was actually permitted to delete other accounts. The UI's absence of a "delete user" button was functioning as the *only* access control, which is not access control at all — it's security through obscurity, defeated the moment the documentation was found.

## Impact
- **Unauthorized administrative action:** a standard, unprivileged user was able to delete another user's account entirely.
- **Full attack surface disclosure:** API documentation reveals every endpoint, parameter, and method the backend supports, dramatically shortening the reconnaissance phase of any further attack.
- **Scalability of the issue:** this pattern generalizes — any endpoint discoverable only through documentation and lacking server-side authorization is exploitable the same way, including potentially far more damaging operations (privilege changes, data exports, account takeovers).

## Remediation
- **Enforce authorization server-side on every endpoint**, regardless of whether the frontend exposes a path to reach it. The UI must never be treated as an access control boundary.
- **Restrict access to API documentation.** Internal/admin-only API specs should require authentication, be hosted on a separate internal-only host, or be excluded entirely from production deployments.
- **Never rely on `robots.txt` or lack of UI linkage for confidentiality** — both are explicitly discoverable and `robots.txt` is, ironically, a public index of "interesting" paths.
- **Apply the principle of least privilege** to every API consumer — role-based checks should validate that the authenticated caller is actually authorized to perform the requested operation on the requested resource, independent of how the request was discovered.
- **Audit undocumented/internal endpoints regularly** as part of the API's security review process — if it exists and is reachable, assume it will eventually be found.

## Key Takeaways
- Hiding an endpoint from the frontend UI provides zero security if the backend will still process the request from any authenticated (or unauthenticated) caller.
- `robots.txt` is a reconnaissance goldmine, not a security control — treat any `Disallow` entry as a strong signal of something worth investigating.
- API documentation is a map of attack surface. Exposed docs routinely reveal endpoints with more capability than the application's visible functionality suggests.
- This lab is a clean example of **Broken Object Level Authorization**: the object (another user's account) was deletable by a caller with no business reason to delete it.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.