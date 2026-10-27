# Access Control: Crossing the Boundaries of Trust

## Platform

PortSwigger Web Security Academy

## Difficulty

Apprentice

## Vulnerability Category

Broken Access Control / Privilege Escalation / Insecure Direct Object Reference (IDOR)

## Lab Objective

Exploit missing or ineffective authorization checks to access restricted functionality, retrieve another user's sensitive information, and escalate privileges.

This writeup covers the five Access Control labs included in the Apprentice Server-side vulnerabilities learning path:

- Unprotected admin functionality.
- Unprotected admin functionality with unpredictable URL.
- User role controlled by request parameter.
- User ID controlled by request parameter, with unpredictable user IDs.
- User ID controlled by request parameter with password disclosure.

The objectives include deleting the lab user `carlos`, retrieving his API key, and exploiting administrator password disclosure.

## Vulnerability Overview

An application can correctly identify a user and still fail to restrict what that user can do.

Authentication establishes identity. Access control determines whether that identity is permitted to access a resource or perform an action.

These labs demonstrate failures across three related boundaries:

| Access-Control Boundary | Expected Restriction | Demonstrated Failure |
| --- | --- | --- |
| Administrative functionality | Only authorized administrators can manage users. | An exposed endpoint or modified cookie grants administrative access. |
| Account ownership | Users can access only account data they are permitted to view. | Changing an account identifier exposes another user's information. |
| Privileged account access | Ordinary users cannot obtain administrator credentials. | An account-page authorization flaw discloses the administrator's password. |

The vulnerabilities arise when the server treats knowledge of an endpoint, possession of an identifier, or a client-supplied role value as sufficient permission.

The central testing question is:

> Does the server independently verify that this user is allowed to access this resource and perform this action?

## Methodology

1. **Map the application's functionality.**  
   Explore public pages, account features, navigation links, page source, and requests generated during normal use.

2. **Identify authorization-sensitive inputs.**  
   Look for administrative routes, role-related cookies, account identifiers, and responses containing private information.

3. **Establish a baseline.**  
   Record the behavior of an unauthenticated request or a request made using the supplied ordinary-user account.

4. **Inspect requests with Burp Suite.**  
   Use **Proxy → HTTP history** to locate relevant requests and send them to **Repeater** for controlled modification.

5. **Change one authorization-related value at a time.**  
   Request a discovered administrative endpoint, modify the role cookie, or substitute another user's identifier while preserving the original session.

6. **Verify the actual impact.**  
   Confirm that restricted functionality or another user's data is accessible. Complete the applicable lab objective and check the solved indicator.

The walkthrough describes the documented exploitation process. Request examples use placeholders rather than captured session values or personal lab evidence.

## Exploitation Steps

### 1. Discovering an Unprotected Admin Panel

**Lab:** Unprotected admin functionality

The administrative interface is accessible without an effective authorization check. Its location is disclosed through `robots.txt`.

1. Request `/robots.txt`.
2. Inspect the `Disallow` entry identifying `/administrator-panel`.
3. Navigate directly to `/administrator-panel`.
4. Use the administrative interface to delete `carlos`.
5. Confirm the lab's solved indicator.

**Why it works:** The server exposes administrative functionality without verifying the caller's privileges.

`robots.txt` provides instructions to cooperative crawlers. Listing a route there does not prevent a person or HTTP client from requesting it.

**Security insight:** Removing a link from the interface or discouraging indexing does not protect the underlying endpoint.

### 2. Finding an Admin Panel Hidden Behind an Unpredictable URL

**Lab:** Unprotected admin functionality with unpredictable URL

The application conceals its administrative interface at an unpredictable address, but discloses that address in client-side JavaScript.

1. Open the homepage source using the browser or inspect its response in Burp.
2. Locate the JavaScript containing the administrative URL.
3. Copy the exact path disclosed by the lab instance.
4. Request that path directly.
5. Delete `carlos` through the administrative interface.

**Why it works:** The server relies on the endpoint being difficult to discover instead of enforcing authorization when it is requested.

The browser receives the JavaScript regardless of whether the page displays an administrative link.

**Security insight:** Client-side visibility rules control presentation. Server-side authorization must control access.

### 3. Forging an Administrative Role Cookie

**Lab:** User role controlled by request parameter

The application identifies administrators through a modifiable cookie named `Admin`.

1. Request `/admin` and observe that access is denied.
2. Sign in using the supplied account:

```text
Username: wiener
Password: peter
```

3. Inspect the login response and identify the cookie:

```text
Admin=false
```

4. Change its value to:

```text
Admin=true
```

5. Request `/admin` with the modified cookie.
6. Use the resulting administrative access to delete `carlos`.

The cookie can be changed by intercepting the response before the browser stores it or by modifying the outgoing request.

**Why it works:** The server accepts an attacker-controlled privilege claim as authority.

**Security insight:** A value returned by the server remains untrusted when the client can change it and submit it again.

### 4. Accessing Another User's Account Through a Disclosed GUID

**Lab:** User ID controlled by request parameter, with unpredictable user IDs

Account identifiers use GUIDs, but the application discloses the target user's identifier through public content.

1. Find a blog post written by `carlos`.
2. Follow the author's link and record the user identifier in its URL.
3. Sign in as `wiener:peter`.
4. Open the account page.
5. Replace the account request's `id` value with Carlos's disclosed GUID, retaining Wiener's session.
6. Retrieve Carlos's API key and submit it through the lab's solution interface.

**Why it works:** The account endpoint accepts a user-controlled identifier without checking whether the current user may access the corresponding account.

**Security insight:** An unpredictable identifier can reduce guessing, but it does not establish ownership or permission.

### 5. Escalating Access Through Administrator Password Disclosure

**Lab:** User ID controlled by request parameter with password disclosure

The account page exposes the existing password in a masked input and permits access to another user's account through its identifier.

1. Sign in as `wiener:peter` and open the account page.
2. Change the request's `id` parameter to `administrator`.
3. Inspect the raw response in Burp.
4. Retrieve the administrator's password from the returned account form.
5. Sign in as `administrator` using that password.
6. Delete `carlos` through the administrative interface.

**Why it works:** An account-authorization flaw exposes a privileged user's password, which then enables administrator authentication.

A password input masks characters visually; it does not conceal a value already present in the response.

**Security insight:** Cross-account disclosure can become administrative compromise when the exposed information includes usable credentials.

### Verification

Successful verification depends on the objective of each lab:

| Lab | Expected Completion Evidence |
| --- | --- |
| Unprotected admin functionality | Access to the panel, deletion of `carlos`, and solved indicator. |
| Unpredictable admin URL | Discovery of the route, unauthorized administrative access, and solved indicator. |
| Role-cookie manipulation | Administrative functionality available with the modified cookie and successful deletion. |
| Unpredictable user IDs | Carlos's API key retrieved through Wiener's session and accepted by the lab. |
| Password disclosure | Administrator sign-in using the disclosed password and successful deletion. |

A successful status code alone does not prove an authorization failure. The response content or completed action must demonstrate access beyond the caller's permissions.

## Payloads

These examples illustrate the relevant request modifications. Replace the hostname, session token, discovered path, and GUID with values from the corresponding lab instance.

### Discovering the Administrative Route

```http
GET /robots.txt HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Accessing the Unprotected Admin Panel

```http
GET /administrator-panel HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Requesting the Disclosed Unpredictable Route

`DISCOVERED_ADMIN_PATH` represents the exact path found in the homepage's JavaScript.

```http
GET /DISCOVERED_ADMIN_PATH HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Forging the Administrative Cookie

```http
GET /admin HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=WIENER_SESSION_TOKEN; Admin=true
```

The relevant modification is:

```text
Admin=false → Admin=true
```

Retain any other cookies required by the captured request.

### Substituting Carlos's GUID

```http
GET /my-account?id=CARLOS_GUID HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=WIENER_SESSION_TOKEN
```

The session remains Wiener's; the requested account identifier changes to Carlos's.

### Requesting the Administrator's Account Page

```http
GET /my-account?id=administrator HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=WIENER_SESSION_TOKEN
```

Inspect the response for the existing password value. Any recovered value belongs to that lab instance and should not be treated as a reusable payload.

## Screenshots

<!--
Insert your own lab evidence using the application's Screenshots feature.

Suggested screenshots:
- robots.txt disclosing the administrative route.
- Homepage JavaScript revealing the unpredictable admin URL.
- The modified Admin cookie and accessible administrative interface.
- Carlos's account data returned through Wiener's session.
- Administrator password disclosure, with the password redacted.
- The relevant solved indicators.

Redact session tokens and recovered credentials before publishing.
-->

## Root Cause

The shared root cause is a failure to enforce authorization at the point where a protected resource is accessed or a privileged action is performed.

| Weakness | Underlying Failure |
| --- | --- |
| Exposed administrative endpoint | The server does not restrict administrative functionality to authorized users. |
| Unpredictable administrative URL | Endpoint secrecy substitutes for a permission check. |
| Forgeable role cookie | A client-controlled value determines administrative privileges. |
| User-controlled account identifier | The server retrieves an account without verifying access to that specific account. |
| Password returned in an account form | Sensitive credentials are exposed through an already vulnerable account endpoint. |

The final lab combines two failures: unauthorized account access and disclosure of an existing password.

The response demonstrates that the application can return the password. It does not, by itself, establish the exact database storage implementation.

## Remediation

### Enforce Authorization on the Server

Check permissions before returning protected data or executing privileged actions. Protect the administrative operation itself as well as the page that links to it.

### Deny Access by Default

Require an explicit authorization decision for protected resources. Apply a consistent authorization mechanism across routes and application components.

### Derive Privileges from Trusted State

Resolve the authenticated user's permissions from server-controlled account or session data. Do not grant privileges because a request contains `Admin=true`.

### Check Access to Each Requested Object

For account requests, verify that the authenticated user is permitted to access the selected account.

Where appropriate, derive the account identity from the authenticated session rather than accepting it from a query parameter.

GUIDs can supplement this design, but they cannot replace an ownership or permission check.

### Keep Existing Passwords Out of Responses

Do not prefill account forms with existing passwords or return them through APIs.

Store passwords using an appropriate salted password-hashing scheme and verify submitted passwords against their hashes. Use dedicated password-change and reset workflows.

### Test Authorization Boundaries

Include tests for anonymous users, ordinary users, and administrators. Verify both access to privileged functions and isolation between accounts with the same role.

Log significant authorization failures and administrative actions to support investigation.

## Key Takeaways

- **Authentication does not grant unrestricted access.** A valid session must still be subject to permission checks.

- **Discoverability and authorization are separate concerns.** Hidden routes remain vulnerable when direct requests are accepted.

- **Client-supplied privilege claims require verification.** A modified cookie must not redefine a user's role.

- **Knowing an identifier does not confer ownership.** Account access must remain restricted even when another user's GUID is public.

- **Visual masking does not protect transmitted secrets.** A password value in HTML is available to whoever receives that response.

- **Small authorization failures can combine into greater impact.** Account-data exposure can lead to administrator compromise.

- **Effective controls protect each resource and action.** The decisive check belongs where the server reads private data or performs a privileged operation.

## References

- [PortSwigger — Access Control Vulnerabilities and Privilege Escalation](https://portswigger.net/web-security/access-control)
- [PortSwigger — Apprentice Server-side Vulnerabilities Learning Path](https://portswigger.net/web-security/learning-paths/server-side-vulnerabilities-apprentice)
- [Lab — Unprotected Admin Functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)
- [Lab — Unprotected Admin Functionality with Unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)
- [Lab — User Role Controlled by Request Parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)
- [Lab — User ID Controlled by Request Parameter, with Unpredictable User IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)
- [Lab — User ID Controlled by Request Parameter with Password Disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure)
- [OWASP — Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)