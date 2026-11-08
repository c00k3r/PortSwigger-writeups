# File Upload Vulnerabilities: Remote Code Execution via Polyglot Web Shell Upload

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Unrestricted File Upload via Polyglot File / Insufficient Content Validation (CWE-434, CWE-646)

## Lab Objective
The avatar upload function now validates file *content*, not just the extension, to confirm the upload is a genuine image. Bypass this by crafting a polyglot file — a single file that is simultaneously a valid image and a valid PHP script — to achieve remote code execution and exfiltrate `/home/carlos/secret`.

## Vulnerability Overview
Every prior lab in this series defeated a check performed on the filename or a client-supplied header. This lab closes that gap properly: the server inspects the file's actual bytes to confirm it's a structurally valid image before accepting it — a meaningfully stronger control than anything in the earlier labs, since it can't be fooled by metadata alone.

The weakness here is more subtle: **image file formats have unused, writable fields that have no effect on whether the file renders as a valid image.** JPEG's Exif metadata standard includes arbitrary, freeform fields — such as the `Comment` field — explicitly designed to hold user-supplied text. An image file is still a perfectly valid, perfectly renderable JPEG whether that comment field is empty, holds a photographer's notes, or holds a complete PHP script. Content validation that checks "is this a structurally valid image" passes either way, because validity as an image and the presence of PHP code in a metadata field are entirely independent properties of the file.

This is the defining property of a **polyglot file**: a single byte sequence that is simultaneously and unambiguously valid under two different format interpretations — here, valid JPEG to an image parser, and valid PHP source to a PHP interpreter, depending entirely on which program reads it.

## Methodology
1. **Confirm content-based validation is in place.** Attempt a raw PHP upload (even with a `.jpg` extension) and observe rejection with a message indicating the file isn't recognized as a genuine image — ruling out the simpler extension/metadata bypasses used in earlier labs.
2. **Identify a writable, inert field within a valid image format.** JPEG's Exif `Comment` tag is a well-known target: freely editable, arbitrary-length, and entirely unvalidated by image-rendering logic.
3. **Use a dedicated metadata tool (ExifTool) to inject the PHP payload** into that field of a genuinely valid image, producing a file that is structurally sound as an image while also containing executable PHP source.
4. **Save the resulting file with a `.php` extension**, combining the polyglot content bypass with the straightforward extension (since content validation, not extension validation, was the control being defeated here).
5. **Upload and invoke the shell directly.**

---

## Exploitation

**Step 1:** Attempt to upload a plain PHP file disguised with a `.jpg` extension. The server rejects it, confirming it inspects file content/structure, not just the name.

**Step 2:** Obtain a genuine, valid JPEG image (any small photo works) and use **ExifTool** to embed a PHP payload into its `Comment` metadata field, saving the result with a `.php` extension:

```bash
exiftool -Comment='<?php echo "START " . file_get_contents("/home/carlos/secret") . " END"; ?>' input.jpg -o polyglot.php
```

This command writes the PHP code into the JPEG's standard Exif `Comment` field and outputs the result as `polyglot.php`. The file's binary structure — its headers, magic bytes, and overall layout — remains that of a fully valid JPEG; only the comment field, which has no bearing on image validity, now contains PHP source.

**Step 3:** Verify the injection succeeded:

```bash
exiftool polyglot.php
```

Confirming the `Comment` field shows the embedded PHP payload.

**Step 4:** Upload `polyglot.php` as the avatar through the normal application flow. The server's content-validation check parses the file, confirms it's a structurally valid JPEG (which it genuinely is), and accepts the upload — the embedded PHP comment is irrelevant to that specific check.

**Step 5:** Locate the uploaded file (via the resulting `<img>` tag or Burp's proxy history) and request it directly:

```
GET /files/avatars/polyglot.php
```

**Step 6:** Because the server executes `.php` files as PHP regardless of their actual content structure, the interpreter parses the file, encounters the embedded PHP code within what it reads as plain text outside of any recognized PHP tags elsewhere in the binary, and executes the payload. The response contains the file's contents wrapped in the `START`/`END` markers used to clearly delimit the exfiltrated secret from surrounding binary noise:

```
START root:x:0:0:root:/root:/bin/bash... END
```

**Step 7:** Submitting the extracted secret solves the lab.

**Why it works:** Content-based image validation and PHP execution are two completely independent processes operating on the same bytes, each using only the portions of the file relevant to their own format. The image validator reads JPEG structure (headers, magic bytes, encoded image data) and ignores the Exif comment entirely, since it has no bearing on whether the file is a valid image. The PHP interpreter, when later asked to execute the file as a script, doesn't care that most of the file is binary JPEG data — it scans for and executes anything between PHP tags, wherever they happen to appear in the byte stream, including inside what the image format considers "just metadata." Both interpretations of the file are simultaneously, genuinely true — that duality is exactly what makes a polyglot file undetectable by content validation that only checks for one format's validity.

## Impact
- **Full remote code execution**, defeating a substantially stronger defensive control than any prior lab in this series — content validation is a real, meaningful barrier, and this lab demonstrates it is still not sufficient on its own.
- **Bypasses the most common "fix" developers reach for after learning about naive upload vulnerabilities** — many teams respond to earlier-stage upload bugs by adding "verify it's a real image" checks, under the mistaken assumption that this closes the vulnerability class entirely. It doesn't.
- **Demonstrates that file format validity and file safety are different properties** — a file can be 100% valid under its claimed format specification and still carry a complete secondary payload, since most format specifications include some amount of freeform, unvalidated space by design (metadata, comments, custom fields).

## Remediation
- **Re-encode uploaded images server-side** rather than trusting the uploaded bytes directly — decoding the image data and writing out a brand-new file using a trusted image library strips all metadata fields (including Exif comments), destroying any embedded payload while preserving the actual visual content.
- **Strip or reject metadata fields entirely** on upload, since legitimate application use cases rarely require preserving arbitrary Exif comments from user-submitted images.
- **Never execute uploaded files as code, regardless of validation strength.** The most robust fix remains unchanged from every prior lab: store uploads outside the webroot, or in a location explicitly configured to serve files as static content only — this makes the question of "is this file secretly also valid PHP" irrelevant, since nothing in that directory executes as a script under any circumstance.
- **Apply strict allowlisting on both extension and MIME type**, as layered defense — though note this alone would not have stopped this specific lab, since the file genuinely is a valid JPEG; execution prevention is the control that actually matters here.
- **Scan uploaded files for anomalies beyond basic format validity** (e.g. unusually large metadata fields, embedded script tags) as an additional detection layer, while recognizing this is a mitigation, not a complete fix.

## Key Takeaways
- Content validation ("is this really an image?") is a genuine security improvement over extension/metadata checks alone, but it answers a narrower question than it might seem to — it confirms format validity, not the absence of a secondary, independently valid payload.
- Polyglot files are possible because most real-world file formats include some amount of specification-sanctioned freeform space (comments, metadata, padding) — understanding a format's structure well enough to know where that space exists is a transferable skill across many polyglot-based attacks (ZIP/JAR polyglots, GIFAR, PDF/HTML polyglots, etc.).
- The only fully reliable fix across this entire upload lab series remains consistent: **never let the upload directory execute code**. Every other defense covered — blacklists, directory restrictions, content validation — has now been individually demonstrated as bypassable; execution prevention is the one control that, done correctly, closes the vulnerability class regardless of how the file itself is crafted.
- ExifTool is a legitimate, widely-used metadata tool with genuine value for both offensive testing and forensic analysis — worth having in your toolkit beyond this specific lab.

## Tools Used
- Burp Suite (Proxy, Repeater)
- ExifTool (metadata injection)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.