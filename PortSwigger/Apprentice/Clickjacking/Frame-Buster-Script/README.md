# Clickjacking with a Frame Buster Script

**Difficulty:** Apprentice  
**Vulnerability:** Clickjacking (UI Redressing)  
**Topic:** Frame Buster Bypass  
**Status:** Solved ✅

## Overview

This lab demonstrates how clickjacking protections implemented through JavaScript frame-busting scripts can be bypassed.

The application prevents itself from being embedded inside an iframe using a frame-buster script. The objective was to bypass this restriction and perform a clickjacking attack to change the victim's email address.

## Methodology

### 1. Login to the application

Logged in using the provided credentials:

- **Username:** `wiener`
- **Password:** `peter`

Navigated to the My Account page and identified the email update functionality.

### 2. Construct the exploit

Opened the exploit server and embedded the target account page inside an iframe.

Used the URL parameter `email` to prepopulate the email update form with an attacker-controlled address.

### 3. Bypass the frame-buster script

Added the following sandbox attribute to the iframe:

```html
sandbox="allow-forms"
```

This prevents JavaScript execution inside the iframe, disabling the frame-busting script while allowing form submissions.

### 4. Align the decoy element

Used CSS positioning to place a decoy element over the underlying Update email button.

Initially set the iframe opacity to `0.1` to identify and align the target button.

After confirming the alignment, reduced the opacity to `0.0001` to make the iframe almost completely transparent.

### 5. Deliver the exploit

Changed the decoy text to **Click me**, stored the exploit, and selected **Deliver exploit to victim**.

The victim's interaction with the decoy element triggered the underlying email update form.

### 6. Verify the result

The lab displayed the **Solved** status.

## Key Takeaways

- Frame-busting scripts are JavaScript-based mechanisms designed to prevent websites from being embedded inside frames.
- The iframe `sandbox` attribute can restrict JavaScript execution.
- The `allow-forms` permission allows form submissions while keeping scripts disabled.
- Clickjacking can be combined with URL parameter-based form prepopulation.
- Client-side frame-busting scripts should not be relied upon as the primary clickjacking defense.
- The CSP `frame-ancestors` directive provides a more robust mechanism for controlling which websites can embed a page.

## Conclusion

Successfully bypassed a JavaScript frame-busting mechanism using the iframe sandbox attribute and performed a clickjacking attack to change the victim's email address.

This lab demonstrated why client-side frame-busting scripts are insufficient protection against clickjacking and highlighted the importance of server-enforced framing restrictions.