# Single-endpoint race conditions

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Race Conditions – Business Logic Vulnerability

## Lab Objective
Exploit a race condition in the application's email change functionality to associate an arbitrary email address with the authenticated account.

The email address `carlos@ginandjuice.shop` has a pending administrator invitation. Successfully claiming this address allows the account to inherit administrator privileges and solve the lab.

## Vulnerability Overview
The application implements an email change feature that allows authenticated users to update their registered email address.

Normally, this process involves submitting a new email address and receiving a confirmation link. The application uses a confirmation token to verify the email change.

However, the implementation contains a race condition. Multiple requests to the same email change endpoint can be processed concurrently, causing the application to handle the email address and confirmation token in an unexpected order.

By exploiting this timing issue, it is possible to associate an email address with an account without completing the intended verification process correctly.

In this lab, the target email address belongs to a pending administrator invitation. Claiming this address allows the authenticated account to inherit the associated administrator privileges.

## Methodology

1. Logged into the application using the provided credentials.
2. Navigated to the account settings and identified the email change functionality.
3. Intercepted the email change request using Burp Suite Proxy.
4. Sent the request to Burp Repeater for further analysis.
5. Examined the request parameters, including the `email` and `csrf` values.
6. Identified that the email change operation was handled by a single endpoint.
7. Created multiple copies of the email change request and configured them to execute simultaneously using Burp Repeater's parallel request functionality.

## Exploitation Steps

### 1. Identify the vulnerable endpoint

Navigated to the account page and initiated an email change.

Intercepted the corresponding HTTP request in Burp Suite Proxy.

The request contained an email parameter and a CSRF token.

### 2. Send the request to Burp Repeater

Forwarded the intercepted request to Burp Repeater.

Identified the following parameters in the request body:

- `email` – Specifies the new email address.
- `csrf` – Contains the CSRF token associated with the authenticated session.

### 3. Prepare the race condition

Created two requests in a Burp Repeater tab group.

**Request 1:** Used an email address controlled through the PortSwigger exploit server.

**Request 2:** Used the target email address:

`carlos@ginandjuice.shop`

Both requests retained the authenticated session cookie and the appropriate CSRF token.

### 4. Execute requests simultaneously

Configured the Repeater tab group to use **Send group (parallel)**.

Sent both requests concurrently to introduce a race condition during the email change process.

Repeated the operation when necessary because successful exploitation depends on request timing and the application's internal processing order.

### 5. Retrieve the confirmation email

After triggering the race condition, accessed the exploit server's Email client.

Retrieved the confirmation email sent to the attacker-controlled email address.

Opened the latest confirmation link in the authenticated lab browser.

### 6. Verify the email association

The race condition allowed the target email address to become associated with the authenticated account without following the intended email verification flow.

Because `carlos@ginandjuice.shop` had a pending administrator invitation, claiming this address resulted in administrator privileges being associated with the account.

The application confirmed that the lab was successfully solved.

## Payloads

**Request 1 – Attacker-controlled email address**

```http
POST /my-account/change-email HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=YOUR-EXPLOIT-SERVER-EMAIL&csrf=YOUR_CSRF_TOKEN
```

**Request 2 – Target email address**

```http
POST /my-account/change-email HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR_SESSION_COOKIE
Content-Type: application/x-www-form-urlencoded

email=carlos@ginandjuice.shop&csrf=YOUR_CSRF_TOKEN
```

**Execution configuration:**

- Tool: Burp Suite Professional – Repeater
- Request grouping: Single tab group
- Execution mode: Send group (parallel)
- Session: Same authenticated session
- CSRF token: Valid token from the authenticated session

*Note: The endpoint and request format above represent the email-change workflow. Replace placeholders with the actual values captured in your lab session.*

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Email Change Request](screenshots/email-change-request.png)

![Parallel Request Execution](screenshots/parallel-request-execution.png)

![Confirmation Email](screenshots/confirmation-email.png)

![Lab Solved](screenshots/lab-solved.png)

## Root Cause

The root cause is improper synchronization of concurrent email change operations.

The application does not correctly maintain the relationship between the requested email address, the confirmation token, and the authenticated account when multiple requests are processed simultaneously.

This creates a race window in which concurrent operations can interfere with each other's state.

The application also relies on an email verification workflow that does not adequately enforce the integrity of the pending email change throughout the entire operation.

## Remediation

Developers should implement the following security controls:

- **Atomic Operations:** Ensure email change operations execute atomically so concurrent requests cannot interfere with one another.
- **Token Binding:** Cryptographically bind confirmation tokens to the specific user account, requested email address, and individual email change operation.
- **Single-use Tokens:** Ensure confirmation tokens can only be used once and are invalidated immediately after successful verification.
- **Concurrency Control:** Implement appropriate locking or transaction mechanisms to prevent conflicting concurrent updates.
- **Server-side Validation:** Revalidate the requested email address and pending verification state immediately before committing the change.
- **Authorization Checks:** Ensure administrator privileges are never granted merely because an email address is associated with an account. Invitations should require a separate, secure authorization and acceptance process.

## References

- PortSwigger Web Security Academy – Race Conditions
- PortSwigger Research – Smashing the State Machine
- OWASP – Race Conditions
- OWASP – Business Logic Security