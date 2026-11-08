# Directory Traversal: Traversal Sequences Stripped Non-Recursively

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Path Traversal (Directory Traversal)

## Lab Objective
Exploit a path traversal vulnerability in the product image display functionality to retrieve the contents of `/etc/passwd`.

The application attempts to prevent directory traversal by removing traversal sequences from user-supplied filenames. However, the filtering mechanism operates non-recursively, allowing an attacker to bypass it using specially crafted nested traversal sequences.

## Vulnerability Overview
Path Traversal is a vulnerability that allows attackers to access files and directories outside the intended application directory by manipulating user-controlled file paths.

Applications often implement input filtering to remove dangerous sequences such as `../`. However, if this filtering is performed only once, attackers can bypass it by constructing nested traversal sequences.

In this lab, the application removes traversal sequences non-recursively. This means that after removing an occurrence of `../`, another traversal sequence can remain within the modified input.

For example, a payload such as `....//` can become `../` after the application removes the inner traversal sequence.

By chaining multiple specially crafted sequences, an attacker can bypass the filter and navigate to sensitive filesystem locations.

## Methodology
1. Reconnaissance of the application and product image functionality.
2. Identifying the image retrieval endpoint and its filename parameter.
3. Intercepting HTTP requests using Burp Suite Proxy.
4. Sending the vulnerable request to Burp Suite Repeater.
5. Testing conventional directory traversal sequences.
6. Identifying the application's non-recursive filtering behavior.
7. Crafting nested traversal sequences to bypass the filtering mechanism.
8. Retrieving the contents of `/etc/passwd`.

## Exploitation Steps

1. Open the lab and navigate to the product listing page.

2. Inspect the product images and identify the endpoint responsible for retrieving them.

3. Open Burp Suite and navigate to **Proxy → HTTP history**.

4. Locate a GET request responsible for loading a product image. Identify the `filename` parameter.

5. Send the request to Burp Suite Repeater.

6. Test a conventional traversal payload:

   `../../../etc/passwd`

   Observe that the application removes traversal sequences, preventing the expected file access.

7. Analyze the filtering behavior. Since traversal sequences are stripped non-recursively, nested sequences can be used to bypass the filter.

8. Replace the filename parameter with the following payload:

   `....//....//....//etc/passwd`

9. Send the modified request using Burp Repeater.

10. Inspect the server response. If successful, the response contains the contents of `/etc/passwd`.

11. Return to the lab and verify that it is marked as solved.

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

**Successful non-recursive filtering bypass:**

```http
GET /image?filename=....//....//....//etc/passwd HTTP/2
Host: YOUR-LAB-ID.web-security-academy.net
```

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The vulnerability exists because the application uses a non-recursive string replacement mechanism to sanitize user-controlled file paths.

Instead of properly validating and canonicalizing the supplied path, the application removes matching traversal sequences only once.

An attacker can exploit this behavior by embedding traversal sequences within other traversal sequences. When the filtering operation removes the inner patterns, the remaining characters form valid directory traversal sequences.

This allows attackers to bypass the filtering mechanism and access files outside the intended directory.

## Remediation

- Avoid using user-controlled input directly in filesystem operations.
- Use strict allow-list validation for filenames.
- Replace user-supplied filenames with server-side identifiers mapped to predefined files.
- Canonicalize file paths before accessing files.
- Verify that resolved paths remain within the intended directory.
- Never rely on simple string replacement to prevent path traversal.
- Implement robust path validation rather than attempting to blacklist individual traversal sequences.
- Restrict filesystem permissions according to the principle of least privilege.

## References

- [PortSwigger Web Security Academy – Path Traversal](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger – Path Traversal: Traversal Sequences Stripped Non-Recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)