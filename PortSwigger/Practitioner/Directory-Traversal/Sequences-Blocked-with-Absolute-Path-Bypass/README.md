# Directory Traversal: Sequences Blocked with Absolute Path Bypass

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Path Traversal (Directory Traversal)

## Lab Objective
Exploit a file path traversal vulnerability in the product image display functionality to retrieve the contents of `/etc/passwd`.

The application blocks common directory traversal sequences but accepts absolute file paths, allowing access to arbitrary files on the server.

## Vulnerability Overview
Path Traversal, also known as Directory Traversal, is a vulnerability that allows an attacker to access files and directories outside the intended directory by manipulating file path parameters.

Applications often use user-controlled input to construct file paths when retrieving resources such as images or documents.

In this lab, the application implements filtering to block directory traversal sequences such as `../`. However, it fails to properly validate absolute file paths.

An attacker can bypass the traversal sequence restriction by supplying an absolute path directly.

Instead of navigating through directories using traversal sequences, the attacker specifies the complete path of the target file.

## Methodology
1. Browse the application and inspect the product image functionality.
2. Identify the image retrieval endpoint and its `filename` parameter.
3. Intercept image requests using Burp Suite Proxy.
4. Analyze how the application handles user-supplied file paths.
5. Test whether directory traversal sequences are blocked.
6. Attempt to bypass the restriction using an absolute file path.
7. Retrieve and inspect the contents of `/etc/passwd`.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Inspect the product images and identify the image retrieval endpoint.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate a GET request responsible for retrieving a product image. The request should contain a parameter similar to:

   `filename=...`

5. Send the request to Burp Repeater.

6. Initially test a conventional traversal payload:

   `../../../etc/passwd`

   Observe that the application blocks traversal sequences.

7. Bypass the filtering mechanism by supplying the absolute path directly:

   `/etc/passwd`

8. Replace the existing filename parameter with the absolute path and send the modified request.

9. Inspect the server response. If successful, the response contains the contents of `/etc/passwd`.

10. Return to the lab and verify that the lab is marked as solved.

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

**Successful absolute path bypass:**

```http
GET /image?filename=/etc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application relies on insufficient input filtering to prevent unauthorized file access.

Although the application blocks common directory traversal sequences such as `../`, it does not properly restrict absolute file paths.

As a result, an attacker can bypass the filtering mechanism by providing an absolute filesystem path.

The underlying issue is the absence of proper server-side file access restrictions and inadequate validation of user-controlled file paths.

## Remediation

- Avoid using user-controlled input directly in filesystem operations.
- Use strict allow-list validation for filenames.
- Map user-supplied identifiers to predefined server-side filenames.
- Canonicalize file paths before performing filesystem operations.
- Verify that resolved paths remain within the intended directory.
- Restrict application filesystem permissions according to the principle of least privilege.
- Reject absolute paths and path traversal sequences where they are not required.
- Implement proper access controls for sensitive server-side files.

## References

- [PortSwigger Web Security Academy – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger – Path Traversal Vulnerabilities](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)