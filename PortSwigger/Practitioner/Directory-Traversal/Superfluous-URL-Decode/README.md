# Directory Traversal: Traversal Sequences Stripped with Superfluous URL-Decode

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Path Traversal (Directory Traversal)

## Lab Objective
Exploit a path traversal vulnerability in the product image display functionality to retrieve the contents of `/etc/passwd`.

The application blocks input containing path traversal sequences and subsequently performs a URL decode operation before using the supplied filename. The objective is to bypass this filtering mechanism using URL encoding and retrieve a sensitive file from the server.

## Vulnerability Overview
Path Traversal vulnerabilities occur when applications use untrusted user input to construct filesystem paths without implementing proper validation.

In this lab, the application attempts to prevent directory traversal by filtering common sequences such as `../`. However, it performs URL decoding after applying the security filter.

This creates an opportunity for an attacker to bypass the validation mechanism using URL encoding.

By encoding characters within the traversal sequence, the attacker can submit a payload that does not initially match the application's filtering rules. When the application subsequently URL-decodes the input, the encoded characters are converted back into traversal characters.

Consequently, the application processes a malicious path that bypassed the initial validation.

## Methodology
1. Reconnaissance of the application and product image functionality.
2. Identifying the image retrieval endpoint and its filename parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the vulnerable request to Burp Suite Repeater.
5. Testing conventional path traversal sequences.
6. Analyzing the application's URL decoding and input filtering behavior.
7. Crafting an encoded traversal payload to bypass the security filter.
8. Retrieving the contents of `/etc/passwd`.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Inspect the product images and identify the endpoint responsible for retrieving them.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate a GET request responsible for loading a product image. Identify the `filename` parameter.

5. Send the request to Burp Suite Repeater.

6. Test a conventional traversal payload:

   `../../../etc/passwd`

   Observe that the application blocks the request because it detects path traversal sequences.

7. Analyze the filtering mechanism. The application performs its security check before URL-decoding the supplied input.

8. Bypass the filter by URL-encoding the dots in the traversal sequence.

9. Use the following payload:

   `..%252f..%252f..%252fetc/passwd`

10. Send the modified request using Burp Repeater.

11. During processing, the application decodes `%25` into `%`, producing `%2f`. Its subsequent URL decoding converts `%2f` into `/`, reconstructing the traversal sequence.

12. Inspect the server response. If successful, the response contains the contents of `/etc/passwd`.

13. Return to the lab and verify that it is marked as solved.

## Payloads

**Original image request (illustrative):**

```http
GET /image?filename=23.jpg HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Conventional traversal attempt:**

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Successful superfluous URL-decoding bypass:**

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application performs security validation before completing URL decoding.

The filtering mechanism checks the encoded input for traversal sequences but fails to account for characters that become dangerous after subsequent decoding operations.

The application effectively trusts input that passes the initial validation, even though later processing transforms it into a malicious filesystem path.

This is an example of improper input canonicalization and inconsistent validation order.

## Remediation

- Decode and canonicalize user input before performing security validation.
- Avoid relying on blacklist-based filtering for path traversal prevention.
- Use strict allow-list validation for filenames.
- Map user-supplied identifiers to predefined server-side filenames.
- Resolve filesystem paths to their canonical form before accessing files.
- Verify that resolved paths remain within the intended directory.
- Avoid multiple or inconsistent URL-decoding operations.
- Restrict application filesystem permissions according to the principle of least privilege.

## References

- [PortSwigger Web Security Academy – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger – Path Traversal: Superfluous URL-Decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)