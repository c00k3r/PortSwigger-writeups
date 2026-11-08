# Directory Traversal: Learning Path Overview

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice to Practitioner (path overview)

## Vulnerability Category
Path Traversal / Directory Traversal (CWE-22)

## Path Objective
This document indexes a progression through PortSwigger's Path Traversal labs — four writeups covering the baseline unprotected case through three distinct, independently broken attempts at sanitising traversal sequences out of user input. Each lab targets the exact same feature (a product-image loader vulnerable to reading `/etc/passwd`), with each successive lab adding one specific defensive mechanism for the previous lab's technique to defeat.

## Writeup Index

| # | Writeup | Technique | Defence Defeated | Level |
|---|---------|-----------|-------------------|-------|
| 1 | [Path Traversal: Escaping the Intended Directory](./path-traversal-escaping-the-intended-directory.md) | Direct `../` traversal | None | Apprentice |
| 2 | [Directory Traversal: Sequences Blocked with Absolute Path Bypass](./directory-traversal-sequences-blocked-with-absolute-path-bypass.md) | Absolute path (`/etc/passwd`) | Relative `../` sequences blocked | Practitioner |
| 3 | [Directory Traversal: Traversal Sequences Stripped Non-Recursively](./directory-traversal-traversal-sequences-stripped-non-recursively.md) | Nested traversal (`....//`) | Single-pass `../` stripping | Practitioner |
| 4 | [Directory Traversal: Extension with Null Byte Bypass](./directory-traversal-extension-with-null-byte-bypass.md) | Null byte (`%00`) truncation | File extension validation | Practitioner |

*(Relative paths above assume each writeup lives alongside this file in the same `Directory Traversal` folder — adjust to match your repo's actual file names if they differ.)*

## Conceptual Progression
As with the rest of this portfolio's grouped series, the sequence here is structured around **which specific assumption the sanitisation logic got wrong**, not simply increasing difficulty for its own sake:

1. **Writeup 1** establishes the baseline: no sanitisation at all. The `filename` parameter is concatenated directly into a filesystem path, and `../../../etc/passwd` climbs straight out of the intended directory, since the OS resolves `..` segments at the point the file is actually opened.
2. **Writeup 2** introduces a filter that blocks relative traversal sequences — but only relative ones. The flaw is an unstated assumption that a "safe" path must be relative to the base directory; an **absolute path** (`/etc/passwd`) bypasses the filter entirely, since it was never designed to recognise an absolute path as a threat in the first place.
3. **Writeup 3** introduces `../` stripping — but implemented as a single, non-recursive pass over the string. The flaw is assuming one removal pass is sufficient; **nesting** the sequence (`....//`) means that after the single pass strips the inner `../`, the surrounding characters collapse back into a working `../`, which was never re-checked.
4. **Writeup 4** introduces extension validation requiring the filename to end in a specific, safe extension (e.g. `.png`) — but validates the *full string*, not the filesystem's actual interpretation of it. A **null byte** (`%00`) injected before the required extension satisfies the string-based check while being silently truncated by the underlying C-based filesystem API, so the file actually opened is the attacker's chosen path, extension check notwithstanding.

## Core Technical Themes Across the Path
- **One root cause, three distinct sanitisation failures.** Every lab traces back to the same underlying flaw — untrusted input reaching a filesystem path with the OS left to resolve it — but each defence fails for a genuinely different reason: an incomplete threat model (Writeup 2), an incomplete algorithm (Writeup 3), and a representation mismatch between validation and execution (Writeup 4).
- **"Blocked" is a prompt to vary the input, not a dead end.** Each successive lab is solved by recognising that a specific, narrow defence implies a specific, narrow blind spot — testing an absolute path, a nested sequence, or an alternate byte-level representation, respectively.
- **Single-pass string operations are a recurring, systemic weakness.** The non-recursive stripping flaw in Writeup 3 is conceptually identical to vulnerabilities seen elsewhere in this portfolio (the blacklist-stripping bypass in the Web Shell Upload via Path Traversal writeup, for instance) — recognising this pattern once makes it easy to spot again in unrelated contexts.
- **Validation and execution operating on different representations of the same string** (Writeup 4's null byte truncation) is the same class of bug seen in the file-upload series' obfuscated-extension lab — a parser differential, not a traversal-specific issue.

## Remediation Summary (applies across the entire path)
- **Canonicalise the path and validate the result**, rather than attempting to detect and strip "bad" substrings from the raw input. Resolve the full, absolute path the input would actually produce, and confirm it falls within the intended base directory before using it.
- **Use a strict allowlist of permitted filenames or identifiers** where feasible, rather than accepting a free-form path component from the client at all — e.g. an opaque product ID mapped server-side to a known filename.
- **Reject absolute paths explicitly**, in addition to relative traversal sequences — both must be handled, not just one.
- **Never rely on a single-pass filter for recursive or nested patterns.** Either loop sanitisation until the input stabilises (no further changes occur) or, preferably, validate the final canonical path instead of iteratively cleaning the raw string.
- **Perform validation on the fully decoded, final string the filesystem will actually receive** — including rejecting null bytes and other non-printable characters outright before any extension or format checks run.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This document indexes a series of writeups covering intentionally vulnerable labs on PortSwigger Web Security Academy, completed for educational purposes. Only test systems you own or have explicit written permission to assess. Individual lab writeups with full exploitation walkthroughs are linked above and included separately in this repository.