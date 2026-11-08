# Directory Traversal: Validation of File Extension with Null Byte Bypass

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Path Traversal (Directory Traversal)

## Lab Objective
Exploit a path traversal vulnerability in the product image display functionality to retrieve the contents of `/etc/passwd`.

The application validates that the supplied filename ends with an expected file extension. The objective is to bypass this extension validation using a null byte injection technique and access a sensitive file outside the intended directory.

## Vulnerability Overview
Path Traversal vulnerabilities occur when applications use user-controlled input to construct filesystem paths without implementing adequate validation.

In this lab, the application checks whether the supplied filename ends with an expected file extension, such as `.png`.

However, the validation mechanism can be bypassed using a null byte (`%00`).

A null byte represents a string terminator in certain programming environments and file-handling APIs. When interpreted by a vulnerable backend component, it can cause the filename to be truncated at the null byte.

An attacker can exploit this behavior by appending the required file extension after the null byte.

For example:

`../../../etc/passwd%00.png`

The application may validate the filename because it ends with `.png`, while the underlying file-handling operation interprets the path as `/etc/passwd`.

This allows an attacker to bypass file extension validation and retrieve arbitrary files.

## Methodology
1. Reconnaissance of the application and product image functionality.
2. Identifying the image retrieval endpoint and its filename parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the vulnerable request to Burp Suite Repeater.
5. Analyzing the application's file extension validation mechanism.
6. Testing whether the application accepts only filenames ending with the expected extension.
7. Crafting a null byte injection payload to bypass extension validation.
8. Retrieving the contents of `/etc/passwd`.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Inspect the product images and identify the endpoint responsible for retrieving them.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate a GET request responsible for loading a product image. Identify the parameter containing the filename.

5. Send the request to Burp Suite Repeater.

6. Inspect the original filename parameter and identify the expected file extension used by the application.

7. Test a conventional path traversal payload:

   `../../../etc/passwd`

   Observe that the application rejects the request because the filename does not end with the expected extension.

8. Analyze the validation mechanism and identify that the application checks the filename extension.

9. Bypass the extension validation by inserting a URL-encoded null byte (`%00`) before the required extension.

10. Use the following payload:

   `../../../etc/passwd%00.png`

11. Replace the original filename parameter with the crafted payload and send the request using Burp Repeater.

12. The application validates the `.png` extension, but the vulnerable file-handling operation truncates the filename at the null byte.

13. Inspect the server response. If successful, the response contains the contents of `/etc/passwd`.

14. Return to the lab and verify that it is marked as solved.

## Payloads

**Original image request (illustrative):**

```http
GET /image?filename=23.png HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Conventional traversal attempt:**

```http
GET /image?filename=../../../etc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Successful null byte bypass:**

```http
GET /image?filename=../../../etc/passwd%00.png HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

**Note:** Replace `.png` with the actual extension enforced by the application, and adjust the traversal depth according to the observed directory structure.

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application relies on file extension validation without properly accounting for how null bytes are handled by downstream filesystem operations.

The application validates the complete user-supplied filename, including the expected extension, but a vulnerable backend component interprets the null byte as a string terminator.

This creates a discrepancy between the filename validated by the application and the filename actually accessed by the filesystem operation.

Consequently, an attacker can bypass extension restrictions and access files outside the intended directory.

## Remediation

- Avoid using user-controlled input directly in filesystem operations.
- Use strict allow-list validation for filenames and extensions.
- Reject null bytes and other invalid control characters in user input.
- Canonicalize filesystem paths before accessing files.
- Verify that resolved paths remain within the intended directory.
- Never rely exclusively on filename extension checks for security.
- Use safe filesystem APIs that handle filenames consistently.
- Restrict application filesystem permissions according to the principle of least privilege.

## References

- [PortSwigger Web Security Academy – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger – Path Traversal: Null Byte Bypass](https://portswigger.net/web-security/file-path-traversal/lab-null-byte-bypass)