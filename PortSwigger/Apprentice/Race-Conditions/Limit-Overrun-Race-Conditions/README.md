# Race Conditions: Limit Overrun Race Conditions

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
Race Conditions – Limit Overrun

## Lab Objective
Exploit a race condition in the application's purchasing flow to purchase a **Lightweight L33t Leather Jacket** for an unintended price.

The application contains a race condition that allows multiple concurrent requests to interact with the same purchasing operation before the relevant state is properly updated.

The objective is to identify and exploit this timing issue using Burp Suite Repeater.

**Credentials:**
- Username: `wiener`
- Password: `peter`

## Vulnerability Overview
A race condition occurs when an application processes multiple requests concurrently and the final result depends on the order or timing in which those requests execute.

A limit overrun race condition occurs when an application enforces a restriction, such as a purchase limit, balance check, or inventory limit, but fails to handle simultaneous requests correctly.

For example, consider a purchasing operation:

1. The application checks whether a user has sufficient balance.
2. The application processes the purchase.
3. The application updates the user's balance.

If multiple purchase requests arrive simultaneously, they may all pass the balance check before the balance is updated.

This creates a race window in which multiple operations can succeed even though only one should have been permitted.

## Methodology

1. **Reconnaissance:** Logged into the application and examined the product purchasing functionality.
2. **Request interception:** Used Burp Suite Proxy to intercept and inspect the relevant purchasing requests.
3. **Request analysis:** Examined the HTTP request responsible for initiating the purchase and identified the relevant parameters.
4. **Race condition testing:** Sent multiple requests concurrently to investigate whether the application correctly enforced its purchasing restrictions.
5. **Burp Repeater configuration:** Used Burp Repeater to group multiple requests and execute them in parallel.
6. **Response analysis:** Compared the responses and application state to determine whether concurrent requests bypassed the intended restriction.

## Exploitation Steps

1. Opened the lab and logged into the application using the provided credentials:
   - Username: `wiener`
   - Password: `peter`

2. Navigated through the online shop and located the **Lightweight L33t Leather Jacket**.

3. Intercepted the purchasing request using Burp Suite Proxy.

4. Sent the captured request to Burp Repeater for further testing.

5. Examined the request and identified the operation responsible for purchasing the product.

6. Created multiple copies of the purchasing request in Burp Repeater.

7. Grouped the requests and configured them to execute concurrently using Burp Repeater's parallel request-sending functionality.

8. Executed the grouped requests to exploit the race window in the application's purchasing flow.

9. Examined the responses and verified whether the application processed multiple operations before correctly enforcing its purchasing restrictions.

10. Successfully exploited the race condition and purchased the **Lightweight L33t Leather Jacket** for an unintended price.

## Payloads

**Purchasing request:**

The exact HTTP request and request body used in the successful exploit should be added here from Burp Repeater.

```http
POST /<purchase-endpoint> HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

<original-purchase-request-body>
```

**Burp Repeater configuration:**

- Duplicate the original purchasing request.
- Add the requests to a Repeater group.
- Configure the group to send requests in parallel.
- Execute the group and inspect the resulting responses.

*Note: The request above is a structural template, not the exact captured lab request. The endpoint and body need to be replaced with the actual successful request.*

## Root Cause

The root cause is improper synchronization of concurrent purchasing operations.

The application does not adequately protect the purchasing workflow against simultaneous requests.

When multiple requests are processed concurrently, the application may perform validation against an outdated application state before updating the relevant purchasing or account information.

This allows multiple operations to pass checks that should have prevented the unintended purchase.

## Remediation

- Implement atomic transactions for purchasing operations.
- Use appropriate database locking mechanisms when modifying shared resources.
- Ensure balance, inventory, and purchase-limit checks occur atomically with the corresponding updates.
- Apply concurrency control to prevent multiple requests from exploiting the same state.
- Use database transactions with appropriate isolation levels.
- Implement idempotency controls where appropriate to prevent duplicate operations.
- Test critical purchasing workflows against concurrent requests.

## References

- [PortSwigger – Race Conditions](https://portswigger.net/web-security/race-conditions)
- [PortSwigger – Limit Overrun Race Conditions](https://portswigger.net/web-security/race-conditions#limit-overrun-race-conditions)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)# Limit Overrun Race Conditions

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
Race Conditions – Limit Overrun

## Lab Objective
Exploit a race condition in the application's purchasing flow to purchase a **Lightweight L33t Leather Jacket** for an unintended price.

The application contains a race condition that allows multiple concurrent requests to interact with the same purchasing operation before the relevant state is properly updated.

The objective is to identify and exploit this timing issue using Burp Suite Repeater.

**Credentials:**
- Username: `wiener`
- Password: `peter`

## Vulnerability Overview
A race condition occurs when an application processes multiple requests concurrently and the final result depends on the order or timing in which those requests execute.

A limit overrun race condition occurs when an application enforces a restriction, such as a purchase limit, balance check, or inventory limit, but fails to handle simultaneous requests correctly.

For example, consider a purchasing operation:

1. The application checks whether a user has sufficient balance.
2. The application processes the purchase.
3. The application updates the user's balance.

If multiple purchase requests arrive simultaneously, they may all pass the balance check before the balance is updated.

This creates a race window in which multiple operations can succeed even though only one should have been permitted.

## Methodology

1. **Reconnaissance:** Logged into the application and examined the product purchasing functionality.
2. **Request interception:** Used Burp Suite Proxy to intercept and inspect the relevant purchasing requests.
3. **Request analysis:** Examined the HTTP request responsible for initiating the purchase and identified the relevant parameters.
4. **Race condition testing:** Sent multiple requests concurrently to investigate whether the application correctly enforced its purchasing restrictions.
5. **Burp Repeater configuration:** Used Burp Repeater to group multiple requests and execute them in parallel.
6. **Response analysis:** Compared the responses and application state to determine whether concurrent requests bypassed the intended restriction.

## Exploitation Steps

1. Opened the lab and logged into the application using the provided credentials:
   - Username: `wiener`
   - Password: `peter`

2. Navigated through the online shop and located the **Lightweight L33t Leather Jacket**.

3. Intercepted the purchasing request using Burp Suite Proxy.

4. Sent the captured request to Burp Repeater for further testing.

5. Examined the request and identified the operation responsible for purchasing the product.

6. Created multiple copies of the purchasing request in Burp Repeater.

7. Grouped the requests and configured them to execute concurrently using Burp Repeater's parallel request-sending functionality.

8. Executed the grouped requests to exploit the race window in the application's purchasing flow.

9. Examined the responses and verified whether the application processed multiple operations before correctly enforcing its purchasing restrictions.

10. Successfully exploited the race condition and purchased the **Lightweight L33t Leather Jacket** for an unintended price.

## Payloads

**Purchasing request:**

The exact HTTP request and request body used in the successful exploit should be added here from Burp Repeater.

```http
POST /<purchase-endpoint> HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=YOUR-SESSION-COOKIE
Content-Type: application/x-www-form-urlencoded

<original-purchase-request-body>
```

**Burp Repeater configuration:**

- Duplicate the original purchasing request.
- Add the requests to a Repeater group.
- Configure the group to send requests in parallel.
- Execute the group and inspect the resulting responses.

*Note: The request above is a structural template, not the exact captured lab request. The endpoint and body need to be replaced with the actual successful request.*

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Purchase Request](screenshots/purchase-request.png)

![Burp Repeater Race Condition](screenshots/burp-repeater-race-condition.png)

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The root cause is improper synchronization of concurrent purchasing operations.

The application does not adequately protect the purchasing workflow against simultaneous requests.

When multiple requests are processed concurrently, the application may perform validation against an outdated application state before updating the relevant purchasing or account information.

This allows multiple operations to pass checks that should have prevented the unintended purchase.

## Remediation

- Implement atomic transactions for purchasing operations.
- Use appropriate database locking mechanisms when modifying shared resources.
- Ensure balance, inventory, and purchase-limit checks occur atomically with the corresponding updates.
- Apply concurrency control to prevent multiple requests from exploiting the same state.
- Use database transactions with appropriate isolation levels.
- Implement idempotency controls where appropriate to prevent duplicate operations.
- Test critical purchasing workflows against concurrent requests.

## References

- [PortSwigger – Race Conditions](https://portswigger.net/web-security/race-conditions)
- [PortSwigger – Limit Overrun Race Conditions](https://portswigger.net/web-security/race-conditions#limit-overrun-race-conditions)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)