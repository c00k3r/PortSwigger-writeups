# Lab: Exploiting Clickjacking Vulnerability to Trigger DOM-Based XSS

**Difficulty:** Practitioner  
**Topic:** Clickjacking, DOM-Based XSS

## Objective
Exploit a clickjacking vulnerability to trick a victim into clicking a hidden button that triggers a DOM-based XSS payload and executes the JavaScript `print()` function.

## Vulnerability Explanation
This lab combines two vulnerabilities:

- **Clickjacking:** An attacker embeds a legitimate website inside an invisible iframe and overlays it with misleading content to trick users into clicking hidden interface elements.
- **DOM-Based XSS:** The application processes attacker-controlled URL parameters in the browser's DOM, allowing injected JavaScript to execute.

By combining these vulnerabilities, an attacker can make a victim unknowingly trigger JavaScript through a seemingly harmless click.

## Exploitation Steps

### 1. Identify the vulnerable functionality
The feedback page accepts URL parameters such as `name`, `email`, `subject`, and `message`.

The `name` parameter can be manipulated to inject an HTML payload.

### 2. Craft the XSS payload
The following payload was used in the `name` parameter:

```html
<img src=1 onerror=print()>
```

The invalid image source triggers the `onerror` event, which executes the JavaScript `print()` function.

### 3. Construct the clickjacking page
An iframe was used to embed the vulnerable feedback page. The iframe was positioned and made transparent, while a decoy element was placed over the target button.

The URL parameters were populated with the XSS payload so that clicking the target button would trigger the vulnerable functionality.

### 4. Align the decoy element
The iframe's position and dimensions were adjusted to align the decoy text with the target button.

The iframe opacity was temporarily set to `0.1` to make alignment easier.

### 5. Verify the exploit
After aligning the elements, clicking the decoy text triggered the browser's print dialog, confirming that the JavaScript payload had executed.

### 6. Deliver the exploit
The iframe opacity was reduced to `0.0001`, and the decoy text was changed to `Click me`.

The exploit was stored on the Exploit Server and delivered to the victim.

## Result
The lab was successfully solved by combining clickjacking with DOM-based XSS. The victim's interaction with the decoy element triggered the hidden functionality and executed the `print()` function.

## Key Takeaways

- Clickjacking can be combined with other client-side vulnerabilities to increase their impact.
- URL parameters should never be treated as trusted HTML.
- DOM-based XSS occurs when client-side JavaScript processes untrusted input unsafely.
- Frame protection mechanisms help prevent unauthorized embedding of sensitive pages.
- Proper output encoding and safe DOM manipulation help prevent DOM-based XSS.