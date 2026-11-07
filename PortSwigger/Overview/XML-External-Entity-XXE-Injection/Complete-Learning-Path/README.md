# XXE Injection: Complete Learning Path (Apprentice → Expert)

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice to Expert (full learning path)

## Vulnerability Category
XML External Entity Injection (CWE-611)

## Path Objective
This document indexes a complete progression through PortSwigger's XXE injection learning path — eight labs that build from a single unprotected entity reference to a network-isolated, OS-fingerprinted data exfiltration chain. Each lab removes one assumption the previous lab relied on (visible output, network egress, attacker-hosted infrastructure, even an obvious XML-shaped request), forcing a genuinely different technique rather than a variation of the same payload.

## Lab Index

| # | Lab | Technique | Constraint Removed | Level |
|---|-----|-----------|---------------------|-------|
| 1 | Exploiting XXE using external entities to retrieve files | Direct entity reflection | — (baseline) | Apprentice |
| 2 | Exploiting XXE to perform SSRF attacks | `SYSTEM` URI redirected to `http://` | File read generalized to internal network access | Apprentice |
| 3 | Exploiting XInclude to retrieve files | `xi:include` element, no `DOCTYPE` needed | Full document control | Practitioner |
| 4 | Exploiting XXE via image file upload | Malicious SVG, Apache Batik | Obvious XML-shaped endpoint | Practitioner |
| 5 | Blind XXE with out-of-band interaction | Collaborator DNS/HTTP callback | Reflected response | Practitioner |
| 6 | Blind XXE with OOB interaction via parameter entities | `%`-entities bypass regex filter | Unfiltered entity syntax | Practitioner |
| 7 | Exploiting blind XXE to retrieve data via error messages | Nested entities → forced parse error | Working OOB/network channel | Practitioner |
| 8 | Exploiting XXE to retrieve data by repurposing a local DTD | Hijacking `ISOamso` in `docbookx.dtd` | Any external resource (hosted DTD included) | Expert |

## Conceptual Progression
The sequence is deliberately structured around **what the attacker no longer has**, not just increasing payload complexity:

1. **Lab 1** establishes the core primitive: an XML parser with external entity resolution enabled, and an application that reflects the result. This is as easy as XXE gets — define an entity, reference it, read the response.
2. **Lab 2** shows the same primitive isn't limited to local files — the `SYSTEM` identifier is protocol-agnostic, so pointing it at `http://169.254.169.254/` turns the exact same vulnerability into SSRF against cloud metadata services.
3. **Lab 3** removes the assumption that the attacker controls the whole XML document. When only a single field can be injected, a `DOCTYPE` can't be defined — but `XInclude`, a separate part of the XML spec, is valid as a self-contained value and achieves the same file-read outcome.
4. **Lab 4** removes the assumption that the vulnerable endpoint looks like XML at all. SVG is XML under the hood; any image/document processing library (here, Apache Batik) that parses an XML-based file format inherits the same risk, hidden behind an "avatar upload" feature.
5. **Lab 5** removes visible output entirely. With nothing reflected, exploitation shifts from "read the response" to "prove the parser acted" — Burp Collaborator turns a DNS lookup and HTTP callback into confirmation of blind exploitation.
6. **Lab 6** removes the assumption that entity syntax is unfiltered. A blocklist tuned to catch `<!ENTITY name SYSTEM ...>` doesn't necessarily catch the structurally distinct parameter-entity form (`<!ENTITY % name SYSTEM ...>`), proving that pattern-based filtering is not equivalent to disabling the underlying feature.
7. **Lab 7** removes network egress as a usable channel entirely. With no response reflection and no OOB interaction possible, data still leaks by engineering a parsing error — nesting entities so the target file's contents become part of an intentionally invalid file path, which the parser then dumps into its own error message.
8. **Lab 8** removes the last remaining external dependency: an attacker-hosted DTD. By redefining a parameter entity (`ISOamso`) known to be declared inside a DTD that already exists on the server's filesystem (`/usr/share/yelp/dtd/docbookx.dtd`, common on GNOME-based Linux), the same error-based exfiltration chain from Lab 7 runs entirely using resources already present on the target — no network interaction of any kind required.

## Core Technical Themes Across the Path
- **One root cause, many surfaces.** Every lab traces back to the same misconfiguration — DTD and external entity processing enabled by default in most XML parsers. The labs vary the *delivery mechanism* (raw XML body, single field, SVG file) and the *exfiltration channel* (reflection, SSRF, OOB callback, error message), never the underlying flaw.
- **Parameter entities vs. general entities.** The distinction between `&name;` (general, usable in the document body) and `%name;` (parameter, usable only within the DTD) is the hinge the second half of the path turns on — both for bypassing filters (Lab 6) and for building the nested, error-triggering chains (Labs 7–8).
- **Numeric character references as a delay mechanism.** `&#x25;` (and its doubly-encoded form, `&#x26;#x25;`) exist to prevent the parser from evaluating a `%` prematurely when entities are nested inside other entity definitions — understanding *why* each level of encoding is needed, not just reproducing it, is what separates genuine comprehension from payload reuse.
- **Verbose error messages as a data channel.** Labs 7 and 8 both depend on the parser including an invalid value directly in its exception message — a theme that recurs across this portfolio's API testing writeups as well, reinforcing that detailed error output is a security-relevant signal, not just a debugging convenience.
- **Network isolation is not a complete mitigation.** Several labs (5, 6, 8) are explicitly constructed to prove that blocking OOB interaction, filtering entity syntax, or restricting egress does not fix XXE — only disabling DTD/external entity processing at the parser level does.

## Remediation Summary (applies across the entire path)
- **Disable DTD processing entirely** wherever the application doesn't genuinely require it — the single fix that closes every variant in this path simultaneously.
- **Disable external entity resolution and XInclude processing independently** if DTDs must remain supported — these are separate parser features, and disabling one does not disable the other (demonstrated explicitly in Lab 3).
- **Never rely on input filtering/blocklisting as the primary defense** — Lab 6 shows a filter targeting one entity syntax is trivially bypassed by a structurally different, equally valid one.
- **Suppress verbose parser error messages in production** — defense-in-depth against Labs 7 and 8's exfiltration technique, independent of the primary parser fix.
- **Don't rely on network egress filtering alone** — Lab 8 demonstrates that local, pre-existing filesystem resources can fully substitute for attacker-hosted infrastructure.
- **Review every XML-processing code path, including third-party libraries** used for file/image/document handling (Lab 4) — not just endpoints that obviously advertise `Content-Type: application/xml`.

## Tools Used
- Burp Suite (Proxy, Repeater, Intruder, Collaborator client)
- PortSwigger exploit server (for hosted-DTD labs)
- Text editor (for crafting malicious SVG/DTD payloads)
- Web browser

## Disclaimer
This document indexes a series of writeups covering intentionally vulnerable labs on PortSwigger Web Security Academy, completed for educational purposes. Only test systems you own or have explicit written permission to assess. Individual lab writeups with full exploitation walkthroughs are linked/included separately in this repository.