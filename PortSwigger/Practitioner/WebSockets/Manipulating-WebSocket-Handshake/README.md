# Manipulating the WebSocket Handshake to Exploit Vulnerabilities

## Lab Description
This lab demonstrates how a flawed XSS filter can be bypassed through WebSocket communication.

The application contains a live chat feature implemented using WebSockets. It uses an aggressive but flawed XSS filter to prevent malicious input.

The objective was to exploit the filtering mechanism by sending a specially crafted WebSocket message that triggers an `alert()` popup in the support agent's browser.

## Vulnerability
**Cross-Site Scripting (XSS) through WebSocket communication**

The application relies on an ineffective XSS filtering mechanism. Although it attempts to block malicious input, its filtering can be bypassed, allowing JavaScript to execute when a manipulated WebSocket message is processed.

## Steps Taken
1. Opened the online shop and accessed its live chat functionality.
2. Sent a normal chat message to establish WebSocket communication.
3. Opened Burp Suite and navigated to **Proxy → WebSockets history**.
4. Identified and inspected the WebSocket messages exchanged between the browser and server.
5. Forwarded a relevant WebSocket message to Burp Repeater.
6. Examined how the application's XSS filter handled user-controlled input.
7. Modified the WebSocket message with a crafted XSS payload designed to bypass the flawed filter.
8. Sent the manipulated message and triggered JavaScript execution in the support agent's browser.
9. Verified that the `alert()` popup appeared, successfully solving the lab.

## Key Takeaways
- WebSocket messages can be intercepted and manipulated using Burp Suite.
- XSS vulnerabilities can exist in real-time communication features.
- Aggressive input filtering does not guarantee protection against XSS.
- Security testing should include attempts to bypass input filters.
- Applications should use context-aware output encoding instead of relying solely on blacklist-based filtering.

## Tools Used
- Burp Suite Professional
- Burp Proxy – WebSockets history
- Burp Repeater
- PortSwigger Web Security Academy