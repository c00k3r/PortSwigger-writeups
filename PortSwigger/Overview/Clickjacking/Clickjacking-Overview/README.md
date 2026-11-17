# Clickjacking – Learning Path Overview

## Introduction

Clickjacking, also known as UI redressing, is a web application vulnerability in which an attacker tricks users into clicking elements on a legitimate website without their knowledge.

The attacker typically embeds the target website inside an invisible or transparent HTML iframe and overlays it with misleading content. When the victim interacts with the visible decoy element, their click is actually directed toward an underlying element on the legitimate website.

This can allow attackers to trigger sensitive actions using the victim's authenticated session.

## Core Concepts

### 1. How Clickjacking Works
- The attacker embeds a target website using an HTML `<iframe>`.
- CSS properties such as `opacity`, `position`, `z-index`, `width`, and `height` control the iframe's visibility and placement.
- Misleading elements are positioned over sensitive buttons or interface components.
- The victim unknowingly interacts with the hidden website.

### 2. Basic Clickjacking
A basic clickjacking attack uses a transparent iframe and a decoy element to trick users into performing an unintended action, such as deleting an account.

### 3. CSRF Token Protection
Clickjacking can still be effective against applications that use CSRF tokens. This is because the victim interacts directly with the legitimate application through an authenticated iframe, causing the browser to submit a valid request.

### 4. Clickjacking with Prefilled Form Input
Attackers can manipulate URL parameters to prepopulate form fields with attacker-controlled values. Combined with clickjacking, this can trick users into submitting unintended information.

### 5. Frame Busting Scripts
Some applications use JavaScript frame-busting techniques to prevent their pages from being embedded inside iframes.

However, certain implementations can be bypassed, allowing an attacker to frame the target page.

### 6. Clickjacking with DOM-Based XSS
Clickjacking can be combined with DOM-based Cross-Site Scripting (XSS). An attacker can trick a victim into clicking a hidden interface element that triggers JavaScript execution.

### 7. Multistep Clickjacking
Some attacks require multiple user interactions. For example:
- The first click triggers an account deletion action.
- The second click confirms the deletion.

Multiple decoy elements can be positioned to target different interface states.

## Important HTML and CSS Properties

| Property | Purpose |
|---|---|
| `iframe` | Embeds another webpage |
| `opacity` | Controls iframe transparency |
| `position` | Controls element positioning |
| `z-index` | Controls stacking order |
| `top` | Sets vertical positioning |
| `left` | Sets horizontal positioning |
| `width` | Controls iframe width |
| `height` | Controls iframe height |

## Common Tools and Techniques

- Burp Suite for inspecting HTTP requests and application behavior.
- Browser developer tools for examining page structure and positioning.
- HTML and CSS for constructing clickjacking proof-of-concepts.
- Exploit Server for hosting and delivering crafted HTML pages.
- URL parameter manipulation for prepopulating form fields.

## Mitigation Techniques

1. **X-Frame-Options:** Restricts whether a webpage can be embedded inside an iframe.
2. **Content Security Policy:** The `frame-ancestors` directive controls which origins are allowed to embed a webpage.
3. **Frame Busting:** JavaScript-based techniques can attempt to prevent unauthorized framing, although browser-enforced protections are preferable.
4. **User Interaction Protections:** Sensitive operations should use appropriate authorization and confirmation mechanisms rather than relying on UI-level protections alone.

## Key Takeaways

- Clickjacking exploits the way browsers handle layered web content and user interactions.
- Transparent iframes are commonly used to conceal legitimate website elements.
- Authentication and CSRF tokens do not inherently prevent clickjacking.
- Clickjacking can be combined with other vulnerabilities, including DOM-based XSS.
- Multistep attacks can target multiple interface states.
- Proper implementation of CSP `frame-ancestors` and X-Frame-Options helps protect applications against unauthorized framing.

## Conclusion

The Clickjacking learning path demonstrates how seemingly simple browser features such as iframes, CSS positioning, and transparency can be abused to manipulate user interactions.

Understanding these techniques is important when assessing web application security, particularly when testing sensitive actions and identifying opportunities to chain client-side vulnerabilities.