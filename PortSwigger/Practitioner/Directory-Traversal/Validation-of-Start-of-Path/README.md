# Directory Traversal: Validation of Start of Path

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Path Traversal (Directory Traversal)

## Lab Objective
Exploit a path traversal vulnerability in the product image display functionality to retrieve the contents of `/etc/passwd`.

The application transmits the full file path through a request parameter and validates that the supplied path starts with an expected directory. The objective is to bypass this validation and access a sensitive file outside the permitted directory.

## Vulnerability Overview
Path Traversal vulnerabilities occur when applications use user-controlled input to construct filesystem paths without implementing adequate validation.

In this lab, the application attempts to prevent unauthorized file access by checking whether the supplied filename starts with an expected directory path.

However, the validation only checks the beginning of the supplied string rather than verifying the canonical filesystem location of the requested file.

An attacker can exploit this weakness by constructing a path that begins with the expected directory but subsequently traverses outside it.

This can be achieved using a directory traversal sequence followed by the target file's absolute path components.

As a result, the application may accept a path that passes its prefix validation while ultimately accessing a file outside the intended directory.

## Methodology
1. Reconnaissance of the application and product image functionality.
2. Identifying the image retrieval endpoint and its filename parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the vulnerable request to Burp Suite Repeater.
5. Analyzing the application's file path validation mechanism.
6. Identifying the expected directory prefix.
7. Crafting a path that satisfies the prefix check while escaping the permitted directory.
8. Retrieving the contents of `/etc/passwd`.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Inspect the product images and identify the endpoint responsible for retrieving them.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate a GET request responsible for loading a product image. Identify the parameter containing the full file path.

5. Send the request to Burp Suite Repeater.

6. Inspect the original filename parameter to identify the expected directory prefix used by the application.

7. Analyze the validation mechanism. The application checks whether the supplied path starts with the expected directory but does not properly verify the final resolved filesystem path.

8. Construct a payload that preserves the expected directory prefix and uses traversal sequences to navigate outside it.

9. Use the following payload, assuming the expected directory is `/var/www/images/`:

   `/var/www/images/../../../etc/passwd`

10. Replace the original filename parameter with the crafted path and send the request.

11. Inspect the server response. If successful, the response contains the contents of `/etc/passwd`.

12. Return to the lab and verify that it is marked as solved.

## Payloads

**Original image request (illustrative):**

```http
GET /image?filename=/var/www/images/23.jpg HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Successful path prefix validation bypass (illustrative):**

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Note:** The directory prefix in the payload must match the actual path observed in the original image request.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application relies on a simple string-prefix validation mechanism to restrict filesystem access.

Checking whether a user-supplied path starts with an expected directory does not guarantee that the resolved file is actually located within that directory.

The application fails to canonicalize the supplied path and verify its final location before accessing the file.

Consequently, an attacker can append directory traversal sequences to an otherwise valid path prefix and escape the intended directory.

## Remediation

- Avoid using user-controlled input directly in filesystem operations.
- Use strict allow-list validation for filenames.
- Map user-supplied identifiers to predefined server-side filenames.
- Canonicalize filesystem paths before accessing files.
- Verify that the resolved canonical path remains within the intended directory.
- Use safe path-handling APIs instead of simple string-prefix checks.
- Ensure directory boundary checks account for path separators.
- Restrict application filesystem permissions according to the principle of least privilege.

## References

- [PortSwigger Web Security Academy – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger – Path Traversal: Validation of Start of Path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)