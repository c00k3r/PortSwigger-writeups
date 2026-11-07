# File Upload Vulnerabilities: Web Shell Upload via Obfuscated File Extension

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Unrestricted File Upload via Null Byte Injection (CWE-434, CWE-626)

## Lab Objective
The avatar upload function blacklists certain extensions and only accepts `.jpg`/`.png` files. Bypass this filter using a classic null byte injection technique to upload a PHP web shell and exfiltrate the contents of `/home/carlos/secret`.

## Vulnerability Overview
This lab revisits one of the oldest tricks in file upload exploitation: the **null byte (`%00`) string-termination bypass**. It exploits a discrepancy between two different layers of the application stack that interpret the same filename string differently — the validation logic, written in a modern, memory-safe language where strings are length-prefixed and can contain any byte including null, versus the underlying filesystem/OS APIs, many of which are built on C conventions where a string is considered to end at the first null byte, regardless of what characters follow it.

If the upload validation checks only that the filename *ends with* an allowed extension, a filename like:

```
exploit.php%00.jpg
```

passes that check — the string does end in `.jpg`. But when that same string reaches a lower-level filesystem write operation that treats `\x00` as a terminator, everything from the null byte onward is silently discarded, and the file is actually written to disk as `exploit.php`. Two components inspected the exact same input and reached two different, security-relevant conclusions about what it meant.

## Methodology
1. **Confirm the blacklist exists.** Attempt a direct `.php` upload and observe rejection, establishing that the application validates file extensions in some form.
2. **Probe the validation logic's exact check.** Test whether a double extension like `exploit.php.jpg` passes — if the server inspects the full filename rather than just the final extension, this will likely fail too, narrowing down that a more specific bypass is needed.
3. **Apply null byte injection**, inserting a URL-encoded null byte between the dangerous extension and a trailing, allowed one.
4. **Observe the upload response carefully** — the filename the application reports back (with the null byte and trailing extension already stripped) is the strongest confirmation the technique worked.
5. **Locate and invoke the shell** at its actual, truncated filename.

---

## Exploitation

**Step 1:** Log in as `wiener:peter` and attempt to upload a basic PHP web shell as the avatar:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

named `exploit.php`. The server rejects it, confirming the extension is validated and blacklisted.

**Step 2:** Send the upload request to **Repeater**. Modify the `filename` field in the `Content-Disposition` header to insert a URL-encoded null byte between the malicious extension and a permitted one:

```http
Content-Disposition: form-data; name="avatar"; filename="exploit.php%00.jpg"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**Step 3:** Send the request. The upload succeeds, and critically, the confirmation response refers to the stored file simply as `exploit.php` — not `exploit.php%00.jpg`. This is the key signal: the null byte and everything after it were silently truncated by the time the filename reached the filesystem write, while the earlier validation step (checking for a `.jpg`/`.png` suffix on the full, un-truncated string) had already passed it.

**Step 4:** Request the uploaded file at its actual, truncated path:

```
GET /files/avatars/exploit.php
```

**Step 5:** The PHP executes normally, and the response contains the contents of `/home/carlos/secret`. Submitting this value solves the lab.

**Why it works:** The vulnerability exists because the extension-validation code and the file-saving code don't agree on where the filename "ends." Validation logic almost certainly used a simple suffix check (`str.endsWith(".jpg")`) against the complete string, which the null-byte-containing filename genuinely satisfies — `exploit.php%00.jpg` really does end in `.jpg` as far as that check is concerned. But the underlying file write operation, likely calling into a C-based or C-influenced filesystem API, terminates the string at the first `\x00` it encounters, producing `exploit.php` on disk. Neither component is "wrong" about its own interpretation of the string — the vulnerability is the mismatch between them, with no layer in the pipeline normalizing or rejecting the ambiguous input before it could be exploited.

## Impact
- **Full remote code execution**, consistent with every upload lab in this series — the technique differs, but a confirmed bypass always yields the same ceiling of impact.
- **Illustrates a general class of vulnerability beyond file uploads**: null byte injection historically affected path validation, authentication checks, and command construction across many languages and frameworks — this lab is a concrete, hands-on instance of a much broader "parser differential" bug class.
- **Highlights the danger of validating a different representation of data than the one actually acted upon** — any time two separate components independently interpret the same input string, a mismatch like this becomes possible, not just in file uploads but across validation layers generally.

## Remediation
- **Perform validation and the file-system operation on the same, fully-resolved string**, ideally the same function/library call, so there is no opportunity for two different interpretations to diverge.
- **Reject filenames containing null bytes (or any non-printable characters) outright** before any further processing — this is cheap, unambiguous, and closes this exact bypass regardless of what validation logic follows.
- **Generate filenames server-side rather than trusting any part of the client-supplied name** — the most robust fix across this entire lab series, since it removes the attacker's ability to influence the stored filename at all, null bytes included.
- **Keep underlying language/runtime and library versions current** — many modern language runtimes and frameworks have hardened their string-handling APIs specifically against null byte truncation issues over the years; legacy or misconfigured stacks remain exposed.
- **Validate actual file content** (magic bytes, re-encoding through a trusted image library) as defense-in-depth, independent of filename-based checks entirely.

## Key Takeaways
- Null byte injection is a "classic" technique for a reason — it exploits a structural mismatch between how different programming layers represent strings, not a simple oversight in one piece of validation logic.
- The clearest tell that a bypass succeeded is often in the application's own response — here, the server quietly confirming a different (truncated) filename than the one submitted was the strongest available signal, often more reliable than guessing and checking execution directly.
- Any validation performed against one representation of an input (the full string) while a downstream system acts on a different representation (the C-string-terminated version) is a parser differential — recognizing this pattern generalizes well beyond file uploads to request smuggling, path traversal, and authentication bypass scenarios.
- This completes a clear arc across the file-upload series so far: no validation → client-controlled metadata trusted → directory execution bypassed via traversal → blacklist itself subverted via `.htaccess` → and now, blacklist logic bypassed via a string-representation mismatch. Each lab defeats a progressively more careful defense using a progressively more subtle technique.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.