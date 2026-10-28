# Authentication: When Identity Checks Fall Short

## Platform

PortSwigger Web Security Academy

## Difficulty

Apprentice

## Vulnerability Category

Broken Authentication / Username Enumeration / Password Brute Force / Two-Factor Authentication Bypass

## Lab Objective

Exploit weaknesses in login responses and authentication workflows to gain access to another user's account.

This writeup covers the two Authentication labs included in the Apprentice Server-side vulnerabilities learning path:

- Username enumeration via different responses.
- 2FA simple bypass.

The first lab demonstrates how revealing error messages support targeted password guessing. The second demonstrates how an incomplete authentication workflow can grant access without verification of the second factor.

## Vulnerability Overview

Authentication establishes whether a user has supplied the evidence required to prove their identity. Its security depends on both the strength of that evidence and the application's enforcement of the verification process.

These labs demonstrate two distinct failures:

| Authentication Stage | Expected Behavior | Demonstrated Weakness |
| --- | --- | --- |
| Username and password verification | Failed attempts should not reveal which part of the credentials was valid. | Different error messages disclose whether an account exists. |
| Protection against repeated guessing | The application should resist automated credential guessing. | Candidate passwords can be tested until a valid combination is identified. |
| Two-factor verification | Protected resources should remain unavailable until both checks succeed. | Direct navigation reaches the account page after password verification alone. |

Authentication and authorization are related but separate responsibilities. Authentication verifies identity; authorization determines what the authenticated identity may access.

The central question throughout these labs is:

> Has the server actually completed every required identity check before granting access?

## Methodology

1. **Map the authentication workflow.**  
   Identify the login form, submitted parameters, redirects, verification pages, session cookies, and destination account page.

2. **Capture a baseline request.**  
   Use **Burp Proxy → HTTP history** to inspect a failed login and establish its normal response.

3. **Compare authentication outcomes.**  
   Look for differences in error messages, response lengths, status codes, and redirect destinations.

4. **Test inputs independently.**  
   During username enumeration, keep the password fixed. After identifying a valid username, keep it fixed while testing candidate passwords.

5. **Inspect workflow enforcement.**  
   In the 2FA lab, determine whether protected pages require completion of the verification step or merely a successful password check.

6. **Verify account access.**  
   Confirm the identity displayed on the account page and the lab's solved indicator. A redirect or successful status code alone is insufficient evidence.

The examples below describe reproducible lab behavior. Hostnames, discovered credentials, and session values are placeholders unless explicitly identified as credentials supplied by PortSwigger.

## Exploitation Steps

### 1. Username Enumeration Through Different Login Responses

**Lab:** Username enumeration via different responses

The application returns distinguishable errors for unknown usernames and incorrect passwords, allowing account discovery before password guessing.

#### Identify a Valid Username

1. Submit an invalid login and locate `POST /login` in Burp.
2. Send the request to **Intruder**.
3. Clear unnecessary payload positions and mark only the username.
4. Select **Sniper** and load PortSwigger's supplied candidate usernames.
5. Keep an incorrect password fixed and run the attack.
6. Compare the response bodies.

The meaningful distinction is:

| Response Message | Interpretation |
| --- | --- |
| `Invalid username` | The submitted username was not recognized. |
| `Incorrect password` | The username was recognized, but the password failed. |

#### Identify the Password

1. Fix the discovered username and move the payload position to the password.
2. Load the supplied candidate passwords and run the attack.
3. Investigate the response returning a `302` redirect instead of the usual failed-login response.
4. Sign in with the identified credentials and open the account page.

**Why it works:** Account-existence disclosure narrows the guessing task, while insufficient protection against repeated attempts allows discovery of the password.

### 2. Bypassing the Second Authentication Factor

**Lab:** 2FA simple bypass

The lab supplies the following credentials:

| Account | Username | Password |
| --- | --- | --- |
| Own account | `wiener` | `peter` |
| Target account | `carlos` | `montoya` |

1. Sign in as `wiener`.
2. Retrieve the verification code through the lab's **Email client** and complete the normal login.
3. Note the account-page path: `/my-account`.
4. Log out, then sign in using Carlos's supplied credentials.
5. At the verification-code prompt, navigate directly to `/my-account` without submitting a code.
6. Confirm that Carlos's account page loads and the lab is solved.

**Why it works:** The protected page accepts the session established after password verification without requiring completion of the second-factor check.

The target password is already provided. The bypass consists of skipping verification rather than discovering or guessing the code.

### Verification

| Lab | Expected Completion Evidence |
| --- | --- |
| Username enumeration via different responses | A distinguishable valid-username response, successful password identification, and access to the corresponding account page. |
| 2FA simple bypass | Access to Carlos's account after password submission without completing his verification-code step. |

Confirm the displayed account identity and solved indicator. Response length and status differences help locate interesting results, but they require inspection and confirmation.

## Payloads

Use the original captured requests as the starting point, preserving any required fields and cookies.

### Baseline Login Request

```http
POST /login HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=invalid-user&password=invalid-password
```

### Username Enumeration Configuration

Mark only the username as the Intruder payload position:

```text
username=§CANDIDATE_USERNAME§&password=invalid-password
```

Use the candidate username list linked from the lab description.

The `§` characters represent positions configured inside Burp Intruder. They are not literal characters to send to the application.

### Password Guessing Configuration

After identifying a valid username, move the payload position to the password:

```text
username=DISCOVERED_USERNAME&password=§CANDIDATE_PASSWORD§
```

Replace the username payload list with PortSwigger's supplied candidate passwords.

### Target Login in the 2FA Lab

```http
POST /login HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=carlos&password=montoya
```

### Direct Account Request Before Second-Factor Verification

```http
GET /my-account HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Cookie: session=CARLOS_POST_PASSWORD_SESSION
```

Use the actual cookies issued during Carlos's login, preserving any additional cookies in the captured request.

The relevant sequence is:

```text
Submit valid password
→ Reach verification-code prompt
→ Request /my-account directly
→ Access granted without second-factor verification
```

## Screenshots

<!--
Insert your own lab evidence using the application's Screenshots feature.

Suggested screenshots:
- Intruder configuration with the username payload position.
- The differing login error messages.
- The successful password attempt and resulting redirect.
- The target account page after successful login.
- The 2FA prompt followed by direct access to Carlos's account.
- The relevant solved indicators.

Redact session tokens and discovered passwords before publishing.
-->

## Root Cause

The first lab combines an information leak with ineffective resistance to repeated login attempts. The second fails to enforce completion of the authentication process.

| Weakness | Underlying Failure |
| --- | --- |
| Distinguishable login errors | The application reveals whether a submitted username exists. |
| Repeated password guessing | Login protections do not prevent discovery using the supplied candidate list. |
| Access before 2FA completion | Protected resources do not require a fully verified authentication state. |

For the 2FA bypass, the observed behavior supports a model in which password verification grants access prematurely. The exact internal session implementation is not established by the HTTP responses alone.

A verification page is only part of the user interface. Its protection depends on the server rejecting requests that attempt to proceed without completing it.

## Remediation

### Use Consistent Failed-Login Responses

Return a generic failure message for incorrect credentials.

Keep failure status codes, response structure, and processing times sufficiently consistent to avoid disclosing account existence through alternative signals.

### Resist Automated Credential Guessing

Apply login throttling and monitoring using account-level and network-level signals. Introduce progressive delays or additional verification where appropriate.

Design restrictions carefully so attackers cannot easily lock legitimate users out.

### Enforce Authentication State on the Server

Maintain a clear distinction between password verification and completion of all required factors.

```text
Unauthenticated
→ Password verified; second factor pending
→ Required factors verified
```

Only the final state should grant access to protected application functionality.

### Restrict Partially Authenticated Sessions

While verification is pending, permit only the limited operations needed to complete or cancel authentication.

Apply this restriction consistently to pages and APIs. Changing the requested URL must not bypass the pending verification state.

### Bind Verification to the Correct Login Attempt

Associate second-factor verification with the intended account and authentication transaction. Limit failed attempts and expire challenges appropriately.

### Test the Complete Workflow

Test direct requests to protected endpoints before login, after password submission, and after full authentication.

Include checks for skipped steps, expired challenges, logout, and attempts to reuse incomplete authentication state.

## Key Takeaways

- **Login errors can disclose useful account information.** A small response difference can turn an unknown username into a confirmed target.

- **Enumeration makes password guessing more focused.** Once the username is known, each subsequent attempt tests only the password.

- **A redirect is a clue, not complete proof.** Confirm successful authentication by accessing the expected account.

- **A verification prompt does not prove enforcement.** Protected resources must independently require the completed authentication state.

- **Known passwords are precisely why a second factor matters.** A sound implementation must still prevent access when the additional verification is missing.

- **Authentication is a process with enforceable transitions.** Every protected request must respect the evidence verified so far.

## References

- [PortSwigger — Authentication Vulnerabilities](https://portswigger.net/web-security/authentication)
- [PortSwigger — Apprentice Server-side Vulnerabilities Learning Path](https://portswigger.net/web-security/learning-paths/server-side-vulnerabilities-apprentice)
- [Lab — Username Enumeration via Different Responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)
- [Lab — 2FA Simple Bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)
- [PortSwigger — Getting Started with Burp Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder/getting-started)
- [PortSwigger — Vulnerabilities in Multi-factor Authentication](https://portswigger.net/web-security/authentication/multi-factor)
- [PortSwigger — Securing Authentication Mechanisms](https://portswigger.net/web-security/authentication/securing)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)