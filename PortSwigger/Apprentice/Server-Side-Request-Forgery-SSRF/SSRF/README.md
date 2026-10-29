# SSRF: Turning the Server Against Itself

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
Server-Side Request Forgery (SSRF), CWE-918

## Lab Objective
Both labs share one core idea: abuse a stock-check feature that makes server-side HTTP requests, and redirect those requests to internal resources the attacker can never reach directly. The final goal in each lab is to reach an internal admin interface and delete the user `carlos`.

| # | Lab | Internal Target | Level |
|---|-----|-----------------|-------|
| 1 | Basic SSRF against the local server | The server itself (`localhost`) | Apprentice |
| 2 | Basic SSRF against another back-end system | Hidden host in `192.168.0.0/24` on port 8080 | Apprentice |

## Vulnerability Overview
Server-Side Request Forgery occurs when an application fetches a resource using a URL that the user controls, without validating where that URL actually points. The server becomes a proxy that sits *inside* the trust boundary, so the attacker borrows its network position and its privileges.

The stock-check feature sends a POST request like this:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

stockApi=http%3A%2F%2Fstock.weliketoshop.net%3A8080%2Fproduct%2Fstock%2Fcheck%3FproductId%3D1%26storeId%3D1
```

The `stockApi` parameter holds a complete URL, and the server requests it on the client's behalf. Because the value is fully attacker-controlled, the request can be pointed anywhere the server can reach.

**Why this works:** Many internal services (admin panels, databases, metadata endpoints) rely on network location as their only access control. A request from `localhost` or from the internal subnet is implicitly trusted. SSRF breaks that assumption, since the request really does originate from a trusted machine.

## Methodology
1. **Reconnaissance.** Browse the shop with Burp Suite proxying traffic. Click **Check stock** on a product and inspect the request in **Proxy > HTTP history**. A parameter containing a full URL (`stockApi`) is a strong SSRF indicator.
2. **Baseline.** Send the request to **Repeater** and confirm normal behaviour (`200 OK` with a stock count).
3. **Redirect the request.** Replace the URL with an internal target and compare the response against the baseline.
4. **Enumerate.** If the target is unknown, scan for it (Lab 2).
5. **Escalate to impact.** Use the discovered admin interface to perform the privileged action (deleting `carlos`).

---

## Lab 01: Basic SSRF Against the Local Server

**Goal:** Access the admin interface at `http://localhost/admin` and delete `carlos`.

**Analysis:** The admin panel is not exposed to external users, but it trusts requests coming from the local machine. The stock-check function runs on that same machine, so it can reach it.

### Exploitation

**Step 1:** Intercept the stock-check request and send it to Repeater. Change the `stockApi` value to the admin panel:

```http
stockApi=http://localhost/admin
```

**Step 2:** Send the request. The response contains the admin interface HTML, which proves the SSRF works. Reading the HTML reveals the delete link:

```html
<a href="/admin/delete?username=carlos">Delete</a>
```

**Step 3:** Call the delete endpoint through the same SSRF channel:

```http
stockApi=http://localhost/admin/delete?username=carlos
```

**Step 4:** The request succeeds and the lab is marked as solved.

**Why it works:** The server fetches `http://localhost/admin` from itself, so the admin application sees a loopback request and skips authentication. The attacker never needs credentials.

---

## Lab 02: Basic SSRF Against Another Back-End System

**Goal:** Use the stock-check feature to scan the internal range `192.168.0.X` on port 8080, find the admin interface, and delete `carlos`.

**Analysis:** Unlike Lab 1, the target is not on the same host, and its exact IP is unknown. SSRF here works as an **internal network scanner**: the server can reach hosts that are invisible from the internet, and the response differences reveal which ones are alive.

### Exploitation

**Step 1:** Send the stock-check request to **Intruder**. Set `stockApi` to a template and mark the last octet as the payload position:

```http
stockApi=http://192.168.0.§1§:8080/admin
```

**Step 2:** Configure the payload.

| Setting | Value |
|---------|-------|
| Attack type | Sniper |
| Payload type | Numbers |
| Range | From 1 to 255, step 1 |

**Step 3:** Start the attack and sort results by **Status**. Most addresses return an error or time out, but exactly one returns `200`. That host is the internal admin interface.

**Step 4:** Send the successful request to Repeater and confirm the admin page loads. Then delete the user (replace `X` with the discovered octet):

```http
stockApi=http://192.168.0.X:8080/admin/delete?username=carlos
```

**Step 5:** The lab is solved once `carlos` is deleted.

**Why it works:** The vulnerable server sits inside the private network, so it can route to `192.168.0.0/24`. The status code and response time act as an oracle: a live host with an admin path answers `200`, while dead addresses fail differently.

---

## Impact
- **Access control bypass:** Admin functionality protected only by network location becomes reachable by anyone.
- **Internal reconnaissance:** Attackers can map hosts, open ports, and services behind the firewall.
- **Privileged actions:** As shown, an attacker can perform administrative operations such as deleting users.
- **Real-world escalation:** In cloud environments, SSRF is commonly used to reach metadata services and steal credentials, which can lead to full account compromise.

## Remediation
- **Use an allowlist:** Only permit requests to known, approved hosts and schemes. Blocklists (for example blocking `localhost`) are routinely bypassed.
- **Avoid user-supplied URLs:** Let the client send an identifier (such as `productId`) and let the server build the URL.
- **Resolve and validate:** Resolve the hostname and reject loopback, private, and link-local ranges *after* DNS resolution.
- **Network segmentation:** Restrict which internal systems the application server can reach.
- **Authenticate internal services:** Never treat "request from localhost" or "request from the internal network" as proof of authorisation.
- **Limit responses:** Do not return raw upstream response bodies to the user.

## Key Takeaways
- Any feature that fetches a URL (webhooks, importers, PDF generators, image fetchers) is an SSRF candidate.
- SSRF turns the server into a **proxy inside the trust boundary**.
- Network location is not authentication.
- Response status codes and timing can act as an oracle for scanning internal networks.

## Tools Used
- Burp Suite (Proxy, Repeater, Intruder)
- Web browser

## Disclaimer
This writeup documents intentionally vulnerable labs on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.