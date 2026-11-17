# Lab: Multistep Clickjacking

**Difficulty:** Practitioner  
**Topic:** Clickjacking, CSRF Protection, UI Redressing

## Objective
Exploit a clickjacking vulnerability to trick a victim into deleting their account by making them perform two clicks:

1. Click the hidden **Delete account** button.
2. Click the **Yes** button in the confirmation dialog.

## Vulnerability Explanation

Clickjacking, also known as UI redressing, is an attack in which an attacker embeds a legitimate website inside an invisible iframe and overlays it with misleading content.

In this lab, the account deletion functionality is protected by a CSRF token and an additional confirmation dialog. However, these protections do not prevent an attacker from tricking a user into interacting with the hidden interface.

The attack requires two separate interactions because the confirmation dialog appears only after the initial deletion request.

## Exploitation Steps

### 1. Access the target application

- Open the lab and log in using the provided credentials:
  - Username: `wiener`
  - Password: `peter`
- Navigate to the My Account page.
- Identify the **Delete account** button.

### 2. Create the clickjacking exploit

Open the Exploit Server and embed the target account page inside an iframe.

The iframe was configured with:
- A fixed width and height.
- Absolute positioning.
- Low opacity to make the target interface nearly invisible.
- A higher `z-index` to place it above the decoy elements.

Two decoy elements were created:
- `Click me first`
- `Click me next`

### 3. Align the first click

The first decoy element was positioned over the hidden **Delete account** button.

The iframe opacity was temporarily set to `0.1` to make alignment easier.

After adjusting the `top` and `left` properties, the cursor changed to a hand when hovering over the decoy text, confirming that it was aligned with the target button.

### 4. Align the second click

Clicking the first decoy element triggered the hidden Delete account button and opened the confirmation dialog.

The second decoy element was then positioned over the **Yes** button in the confirmation dialog.

Its position was adjusted until the cursor indicated that it was correctly aligned.

### 5. Finalize the exploit

After confirming both alignments:

- Changed the iframe opacity to `0.0001`.
- Changed the decoy text to `Click me first` and `Click me next`.
- Stored the updated exploit on the Exploit Server.

### 6. Deliver the exploit

Clicked **Deliver exploit to victim**.

The victim's two interactions triggered the hidden account deletion button and subsequently confirmed the deletion.

## Result

The lab was successfully solved by combining clickjacking with a multistep interaction flow. Both the account deletion action and its confirmation were triggered through carefully positioned decoy elements.

## Key Takeaways

- Clickjacking can exploit legitimate user interactions without directly bypassing application authentication.
- Confirmation dialogs do not necessarily prevent clickjacking.
- Multistep clickjacking requires careful alignment across different interface states.
- CSRF tokens do not inherently protect against clickjacking.
- Appropriate frame restrictions, such as CSP `frame-ancestors` and `X-Frame-Options`, help prevent unauthorized framing.