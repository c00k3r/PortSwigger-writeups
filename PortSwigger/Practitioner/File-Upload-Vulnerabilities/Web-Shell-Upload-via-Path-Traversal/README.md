# File Upload Vulnerabilities: Web Shell Upload via Path Traversal

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Unrestricted File Upload chained with Path Traversal (CWE-434, CWE-22)

## Lab Objective
The avatar upload function is configured so that files stored in its destination directory (`/files/avatars/`) cannot be executed as scripts — a server-level restriction, not just a file-type check. Bypass this by using path traversal in the uploaded filename to write the web shell to a sibling directory where execution *is* permitted, then use it to exfiltrate `/home/carlos/secret`.

## Vulnerability Overview
This lab represents a meaningfully different defensive posture than the earlier Apprentice file-upload labs: instead of relying solely on validating what gets uploaded, the application also hardens *where* uploads are allowed to execute — `/files/avatars/` is explicitly configured to serve files as static content only, regardless of extension. Uploading `exploit.php` there succeeds, but requesting it back returns the raw PHP source as plain text rather than executing it, confirming the directory itself is non-executable by design.

This is a genuinely stronger control than a simple extension blacklist. However, it only protects *that specific directory* — it says nothing about any other directory the web server happens to serve. If the upload mechanism can be tricked into writing the file **outside** the hardened directory via path traversal in the filename, the file lands somewhere the server will execute it normally, and the directory-level restriction is rendered irrelevant.

## Methodology
1. **Confirm the upload succeeds but execution is blocked.** Upload a PHP file through the normal avatar flow, then request it back directly — if the source code is returned as plain text instead of being executed, the storage directory itself is hardened against script execution.
2. **Identify the storage path** from the upload response or the resulting `<img>` tag, confirming the exact directory (`/files/avatars/`) the restriction applies to.
3. **Attempt a direct traversal payload** in the filename field, and observe whether the server simply strips `../` sequences — a common, incomplete defense against path traversal.
4. **Obfuscate the traversal sequence** (URL-encoding the slash) to bypass naive string-based stripping.
5. **Locate the file at its new, traversed location** — outside the hardened directory — and confirm it now executes.
6. **Invoke the shell** to read the target file.

---

## Exploitation

**Step 1:** Log in as `wiener:peter` and upload a basic PHP web shell as the avatar, named `exploit.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**Step 2:** Navigate to the resulting avatar URL directly (e.g. `/files/avatars/exploit.php`). Instead of executing, the response returns the raw PHP source as plain text — confirming the `/files/avatars/` directory is configured to prevent script execution regardless of file extension.

**Step 3:** Send the original upload request to **Repeater**. Locate the `Content-Disposition` header for the file part and modify the `filename` to include a traversal sequence, attempting to write the file one directory up:

```http
Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
```

**Step 4:** Send the request. The response confirms the file was stored as `avatars/exploit.php` — unchanged — indicating the server is actively stripping the literal `../` sequence from the filename before using it, a common but incomplete sanitisation approach (matching and removing the literal string, rather than canonicalising and validating the resulting path).

**Step 5:** Obfuscate the traversal sequence by URL-encoding the forward slash, which survives the literal-string strip since the stripping logic likely only recognises the unencoded `../` pattern:

```http
Content-Disposition: form-data; name="avatar"; filename="..%2fexploit.php"
```

**Step 6:** Send the modified request. The decoded filename now resolves to `../exploit.php` *after* the sanitisation step has already run, writing the file one level above the hardened `avatars/` directory — into `/files/exploit.php`, a path not covered by the execution restriction.

**Step 7:** Request the file at its new, traversed location:

```
GET /files/exploit.php
```

**Step 8:** This time, the PHP executes normally and the response contains the contents of `/home/carlos/secret`. Submit the retrieved value via the lab's solution button to solve the lab.

**Why it works:** Two independent weaknesses combine here. First, the directory-level execution restriction is real but narrowly scoped — it protects one specific path, not the filesystem generally, so escaping that path entirely defeats it. Second, the path traversal filter is itself incomplete: stripping a literal `../` substring is a classic, broken sanitisation pattern, because it only catches the single most obvious encoding of "go up one directory." URL-encoding the slash changes the string's representation without changing its meaning once decoded, and if sanitisation happens before decoding (or isn't reapplied after it), the traversal survives intact through to the filesystem write.

## Impact
- **Full remote code execution**, identical in severity to the simpler upload labs — the extra directory-level hardening here raises the bar for exploitation but doesn't change the ceiling of impact once bypassed.
- **Demonstrates that directory-scoped mitigations are incomplete on their own** — hardening *where* files execute is a reasonable layer of defense, but it's trivially defeated if the attacker can control *where* the file actually gets written.
- **Classic blacklist-based path traversal filtering is insufficient**, exactly as seen in the dedicated Path Traversal lab series earlier in this portfolio — the same single-pass string-stripping weakness recurs here in a completely different vulnerability category, underscoring how narrow, pattern-based fixes fail to generalize.

## Remediation
- **Generate filenames server-side**, never reusing any part of the client-supplied filename for storage — this single fix eliminates the path traversal vector entirely, independent of how well the traversal-stripping logic is written.
- **Canonicalise and validate the final resolved path**, not just the filename string, before writing to disk — compare the resolved absolute path against the intended base directory and reject anything that doesn't match, rather than attempting to detect and strip "bad" substrings.
- **Apply directory-level non-execution as defense-in-depth, not a sole control.** It remains a valuable layer (it would have stopped a simpler attack with no traversal bypass available), but should never be relied upon as the only barrier against RCE.
- **Validate input after full decoding, not before.** Sanitisation routines must operate on the fully decoded value the filesystem will actually receive — filtering a raw, still-encoded string leaves exactly this kind of bypass open.
- **Store uploads outside the webroot entirely** where possible, removing the question of "which directories are executable" altogether.

## Key Takeaways
- Defense-in-depth measures (here, a non-executable upload directory) are genuinely valuable but must be paired with equally rigorous input validation — one strong control plus one weak one still yields a bypass.
- Path traversal filters built around stripping a specific literal substring are a recurring, systemic weak point across entirely different vulnerability classes (seen here, and in the dedicated Path Traversal labs) — always test with an encoded variant before concluding a traversal filter is effective.
- A file upload restriction scoped to "this file won't execute in this directory" says nothing about adjacent directories — always check whether a traversal primitive can relocate the payload outside the restricted scope.
- This lab is a clean illustration of vulnerability chaining: neither the directory hardening bypass nor the traversal filter bypass alone would have been sufficient; the exploit required both in sequence.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.