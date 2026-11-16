# Multi-endpoint Race Conditions

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Race Conditions – Multi-endpoint Race Condition

## Lab Objective
Exploit a race condition in the shopping cart's purchasing workflow to purchase the **Lightweight L33t Leather Jacket**, priced at $1337, despite having insufficient store credit.

The objective was to manipulate the timing of two concurrent requests — adding an item to the cart and checking out — to bypass the application's intended purchasing restrictions.

## Vulnerability Overview
A race condition occurs when an application processes multiple requests concurrently and the final outcome depends on the order or timing of their execution.

In this lab, the shopping cart functionality involves two important endpoints:

- `POST /cart` – Adds products to the shopping cart.
- `POST /cart/checkout` – Processes the checkout and validates the purchase.

The vulnerability arises because the application handles cart modifications and checkout operations in a way that allows concurrent requests to interfere with each other.

By sending both requests simultaneously, it is possible to exploit a timing window between the validation of the cart's contents and the completion of the checkout operation.

This can result in the application processing a purchase that would normally be rejected due to insufficient store credit.

## Methodology

1. Logged into the application using the provided credentials.
2. Purchased a gift card to understand the application's purchasing workflow.
3. Used Burp Suite Proxy HTTP history to identify the relevant cart endpoints.
4. Examined the `GET /cart` request with and without the session cookie to understand how cart state was maintained.
5. Sent the `POST /cart` and `POST /cart/checkout` requests to Burp Repeater.
6. Created a Repeater tab group containing both requests.
7. Benchmarked sequential request execution to understand response timing.
8. Used a homepage request (`GET /`) to warm up the connection and reduce initial connection overhead.
9. Modified the cart request to add the target leather jacket using its product ID.
10. Compared sequential and parallel request execution.

## Exploitation Steps

### 1. Identify the vulnerable endpoints

Inspected Burp Suite Proxy HTTP history and identified:

- `POST /cart`
- `GET /cart`
- `POST /cart/checkout`

The cart was associated with the user's session, indicating that requests using the same session could interact with the same cart state.

### 2. Investigate session-based cart storage

Sent the `GET /cart` request to Burp Repeater.

Compared the response when the session cookie was present and when it was removed.

Without the session cookie, the application returned an empty cart.

This indicated that the shopping cart was maintained server-side and associated with the user's session.

### 3. Benchmark the purchasing requests

Sent `POST /cart` and `POST /cart/checkout` to Burp Repeater and added both requests to a new tab group.

Initially, sent the requests sequentially using a single connection.

Observed that the first request consistently took longer than the second.

Added a `GET /` homepage request at the beginning of the group to warm up the connection.

After warming up the connection, the subsequent requests completed within a much smaller time window.

Removed the homepage request before continuing.

### 4. Modify the cart request

Modified the `POST /cart` request to add the target leather jacket.

The target product ID was `1`.

The modified request body was:

```http
productId=1&redir=PRODUCT&quantity=1
```

The server responded with:

```http
HTTP/2 302 Found
Location: /product?productId=1
```

This indicated that the add-to-cart request had been processed and redirected to the product page.

### 5. Establish the baseline behavior

Tested the requests sequentially.

The checkout operation was rejected because the available store credit was insufficient to purchase the $1337 leather jacket.

This confirmed the application's expected behavior without exploiting the race condition.

### 6. Prepare the race condition

Reset the cart by removing the leather jacket.

Kept one gift card in the cart and added another gift card.

The cart was then prepared with two gift cards before attempting the concurrent requests.

### 7. Execute the parallel requests

Returned to Burp Repeater and used the existing tab group containing:

- `POST /cart` – Adds the leather jacket.
- `POST /cart/checkout` – Processes the purchase.

Changed the group sending mode from sequential execution to **Send group (parallel)**.

Sent both requests simultaneously using Burp Repeater.

This attempted to exploit the race window between cart modification and checkout processing.

### 8. Verify the result

Inspected the response to:

```http
POST /cart/checkout
```

The server returned:

```http
HTTP/2 200 OK
```

Inspected the response HTML and located the target product:

```html
Lightweight "133t" Leather Jacket
```

The response showed:

- Product: Lightweight "133t" Leather Jacket
- Price: $1337.00
- Quantity: 1

The lab subsequently confirmed successful completion.

## Payloads

### Add-to-cart request

```http
POST /cart HTTP/2
Host: <lab-host>
Cookie: session=<session-cookie>
Content-Type: application/x-www-form-urlencoded

productId=1&redir=PRODUCT&quantity=1
```

### Checkout request

```http
POST /cart/checkout HTTP/2
Host: <lab-host>
Cookie: session=<session-cookie>
Content-Type: application/x-www-form-urlencoded

<original checkout request body captured from Burp>
```

**Note:** The original checkout request parameters, headers, and session cookie were retained. Both requests were executed concurrently through Burp Repeater.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Repeater Request Group](screenshots/repeater-race-condition-group.png)

![Modified Add-to-Cart Request](screenshots/modified-cart-request.png)

![Parallel Request Execution](screenshots/parallel-request-execution.png)

![Checkout Response](screenshots/checkout-response.png)

![Lab Solved](screenshots/lab-solved.png)

## Root Cause

The underlying issue is improper synchronization of concurrent operations that modify and validate shared shopping cart state.

The application does not adequately ensure that cart validation and checkout processing operate on a consistent, atomic view of the cart.

When multiple requests are processed concurrently, the application can encounter an inconsistent state between checking the cart's contents and completing the transaction.

This creates a race window that can be exploited to bypass purchasing restrictions.

## Remediation

Developers should implement appropriate concurrency controls around sensitive purchasing operations.

Recommended security measures include:

- **Atomic transactions:** Perform cart validation, balance verification, and order creation within a single database transaction.
- **Database locking:** Use appropriate row-level locks or other concurrency controls to prevent conflicting cart modifications during checkout.
- **Consistent state validation:** Revalidate cart contents and total purchase cost immediately before committing the transaction.
- **Server-side price validation:** Never trust client-controlled product prices, quantities, or cart totals.
- **Concurrency testing:** Test purchasing workflows under simultaneous requests to identify race conditions.
- **Idempotency controls:** Use appropriate idempotency mechanisms to prevent duplicate or inconsistent transaction processing.

## References

- [PortSwigger Web Security Academy – Race Conditions](https://portswigger.net/web-security/race-conditions)
- [PortSwigger Research – Smashing the State Machine](https://portswigger.net/research/smashing-the-state-machine)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)