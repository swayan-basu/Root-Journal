## DVWA Security Testing Writeups (Low Level) — OWASP ZAP & Manual Exploitation

### Lab Environment Setup
* **Target:** Damn Vulnerable Web Application (DVWA v1.0)
* **Target URL:** `http://127.0.0.1:8080`
* **Security Level of the Target:** Low
* **Deployment Environment:** Deployed on DVWA Instant lab container on Debian Linux. Installation done using `snap install instant-lab`.
* **Testing Tools:** OWASP ZAP v2.17.0 (used local intercepting proxy) & Mozilla Developer Tools.

---
## 1. Testing Summary & Methodology

This write-up documents a hands-on web application security assessment performed against the DVWA or Damn vulnerable web application and testing performed to **Low Security Level**.

### Workflow:
1. **Lab Provisioning:** Deployed the DVWA container using instant labs’ docker interface.

2. **Proxy & Interception Setup:** Configured OWASP ZAP as a local HTTP proxy listening on `127.0.0.1:8081`

3. **Reconnaissance & Mapping:** Performed ZAP Manual Explore to map endpoints of the target(`/login.php`, `/index.php`, `/vulnerabilities/`).

4. **Targeted Exploitation:** Tested almost every attack module listed on DVWA web interface except Insecure Captcha as part of practice web testing.

---

## 2. Detailed Testing & Vulnerability Breakdown

### Module 1: Brute Force
* **Endpoint:** `/vulnerabilities/brute/`
* **Severity:** Medium / High

#### Vulnerability Details
The authentication form handles login submissions via HTTP `GET` requests and lacks automated protections such as rate-limiting, account lockouts, or CAPTCHA verification at low security levels. This allows attackers to systematically test credential lists using automated zap fuzzer.

#### Testing Steps
1. Navigated to the Brute Force module (`/vulnerabilities/brute/`)
2. Captured a sample login request in OWASP ZAP to inspect query parameter serialization.
3. Configured the **OWASP ZAP Fuzzer** by selecting the password parameter as the fuzz location and supplying a test wordlist.
4. Executed the automated fuzzing attack and analyzed the resulting response body sizes to identify valid credentials.

#### OWASP ZAP
* **Fuzzed Parameter:** `password` in the `GET` query string.
* **Telemetry Analysis:** Monitored response size variations across multiple rapid requests to pinpoint authentication success conditions

#### Remediation Recommendations
* **Switch to POST Requests:** Transmit sensitive credentials within the body of `POST` requests rather than exposing them in URL query parameters.
* **Implement Rate Limiting & Lockouts:** Apply exponential delays or temporary account lockouts after consecutive failed login attempts.
* **Multi-Factor Authentication:** Require secondary verification mechanisms or CAPTCHA challenges to block automated bot traffic.

#### Result
Successfully demonstrated that the lack of automated defenses enables attackers to execute rapid credential guessing attacks against user accounts.

<img width="1536" height="141" alt="bf-zap" src="https://github.com/user-attachments/assets/c6c48274-9d95-4934-8875-2d0f112db7cb" />
<img width="1391" height="854" alt="bf" src="https://github.com/user-attachments/assets/bb5e6a37-eba3-44ba-9d10-324e47155e68" />


---

### Module 2: Command Injection
* **Endpoint:** `/vulnerabilities/exec/`
* **Vulnerable Parameter:** `ip`
* **Severity:** High / Critical

#### Vulnerability Details
The application implements a network utility ping script that incorporates user-supplied input directly into system shell commands without applying proper sanitization, filtering, or input validation against command-chaining characters (such as `;`, `|`, and `&&`).

#### Testing Steps:
1. Navigated to the Command Injection endpoint (`/vulnerabilities/exec/`).
2. Entered test input (`127.0.0.1;`) into the target IP address input box.
3. Intercepted and inspected the form submission traffic using OWASP ZAP to monitor parameter serialization and request handling.
4. Observed the backend server response reflecting the execution output of the concatenated command sequence.

#### OWASP ZAP
* **Request Method (`POST`):** Captured requests targeting `http://127.0.0.1:8080/vulnerabilities/exec/` returning `200 OK`.
* **Behavioral Response:** The backend system shell executed the appended instructions sequentially alongside the primary ping utility function.

#### Remediation Recommendations
1. Avoid Shell Execution: Use native language built-ins or safe programming APIs instead of passing user inputs directly to operating system shell utilities (e.g., using built-in socket or network libraries).
2. Strict Whitelisting: If system commands must be executed, enforce a strict regex whitelist ensuring only valid IPv4/IPv6 address formats are accepted.
   
#### Result
Successfully demonstrated that unvalidated input parameters allow command chaining, granting attackers arbitrary Remote Code Execution (RCE) capabilities under the web server's user context.

---

### Module 3: Cross-Site Request Forgery (CSRF)
* **Endpoint:**`/vulnerabilities/csrf/`
* **Severity:** Medium / High

#### Vulnerability Details
The password update form lacks anti-CSRF tokens (such as anti-CSRF or synchronizer tokens) and relies solely on predictable cookie-based session management (`PHPSESSID`). Because state-changing actions are handled via standard `GET` requests without validating the request origin, an attacker can trick an authenticated user into executing unauthorized commands.

#### Testing Steps
1. Navigated to the CSRF vulnerability module (`/vulnerabilities/csrf/`).
2. Executed a password change request directly through the interface or via crafted URL query parameters (`password_new=admin&password_conf=admin&Change=Change`).
3. Captured and inspected the request sequence using OWASP ZAP proxy history to verify that session cookies automatically authorize state modification.

#### OWASP ZAP
* **Request Method (`GET`):**
```http
  GET
(http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=admin&password_conf=admin&Change=Change) HTTP/1.1
  Host: 127.0.0.1:8080
  Cookie: security=low
PHPSESSID=1918049c64aedbbb198f608b70675cb7
```

​**Proxy History Log:** Captured consecutive telemetry entries tracking repeated credential update payloads returning a 200 OK status.  

​#### Results:
​* **Application Response:** The server processed the request and displayed the confirmation text: "Password Changed."  
​* **Validation Flaw:** The application successfully modified the account credentials without requiring the user to re-enter their current password or verifying a unique per-session cryptographic token.

​#### Remediation Recommendations:
​* **Implement Anti-CSRF Tokens:** Embed unique, unpredictable, server-validated tokens into all state-changing form submissions.
​* **Require Re-Authentication:** Force users to provide their existing password before allowing credential updates.
​* **Use POST for State Changes:** Restrict sensitive data modifications to POST requests rather than predictable GET parameters.

<img width="1570" height="36" alt="csrf-zap02" src="https://github.com/user-attachments/assets/1f6063b0-3422-4bed-8a5d-1e85d42163f1" />
<img width="813" height="292" alt="csrf-zap" src="https://github.com/user-attachments/assets/b868bfac-4d40-4c2c-bf87-c70a755959c9" />
<img width="1394" height="851" alt="csrf" src="https://github.com/user-attachments/assets/e927a1ea-d447-4e61-bab1-622383599a12" />

---

### Module 4: File Inclusion
* **Endpoint:** `/vulnerabilities/fi/`
* **Vulnerable Parameter:** `page`
* **Severity:** High

#### Vulnerability Details
The application takes user input via the `page` parameter and passes it directly into a file inclusion function (such as `include()` or `require()`) without sufficient validation, filtering, or path whitelisting. This allows attackers to manipulate the parameter to dynamically load arbitrary local files or external resources.

#### Testing Steps
1. Navigated to the File Inclusion endpoint (`/vulnerabilities/fi/`).
2. Observed the default navigation options which pass filenames via the URL query string (e.g., `?page=file1.php`).
3. Tested parameter handling by manually specifying target local files (`?page=file1.php`) directly in the URL bar and monitoring application output.

#### OWASP ZAP
* **Request Method (`GET`):** Captured requests targeting `http://127.0.0.1:8080/vulnerabilities/fi/?page=file1.php` returning `200 OK`.
* **Behavioral Response:** The backend successfully resolved and rendered the script contents dynamically based entirely on the user-controlled parameter value.

#### Remediation Recommendations
* **Enforce Strict Whitelisting:** Use a hardcoded array or map matching allowed keys to explicit file paths rather than accepting arbitrary user input.
* **Disable Remote Inclusion:** Ensure PHP configurations have `allow_url_include` turned off to mitigate remote file inclusion risks.

#### Result
Demonstrated that the application dynamically evaluates and loads files based on unvalidated user input, potentially exposing sensitive source code or system configurations.

<img width="1388" height="672" alt="file inclusion" src="https://github.com/user-attachments/assets/e4d2cea9-d731-433e-8d7d-0182894d9deb" />

---

### Module 5: Unrestricted File Upload
* **Endpoint:** `/vulnerabilities/upload/`
* **Vulnerable Function:** File upload form
* **Severity:** High

#### Vulnerability Details
The application permits users to upload files without restricting file extensions or validating MIME/Content-Type headers on the server side. Additionally, uploaded files are stored in a web-accessible directory (`../../hackable/uploads/`) with execute permissions, allowing arbitrary code execution.

#### Testing Steps:
1. Navigated to the File Upload endpoint (`/vulnerabilities/upload/`).
2. Created a basic proof-of-concept PHP script (`shell.php`).
3. Intercepted and submitted the upload request using OWASP ZAP to inspect the HTTP request parameters.
4. Verified that the server returned a `200 OK` HTTP status and displayed the file save path (`../../hackable/uploads/shell.php`).

#### OWASP ZAP
* **Request Method:** `POST http://0.0.0.0:8080/vulnerabilities/upload/`
* **Content-Type:** `multipart/form-data`
* **Uploaded File:** `filename="shell.php"` with `Content-Type: application/x-php`
* **Response:** File accepted and written to disk without restriction

#### Remediation Recommendations
* **Extension Whitelisting:** Validate file extensions on the server side against a strict whitelist of allowed safe types (e.g., `.png`, `.jpg`, `.pdf`).
* **Content Validation:** Inspect file headers and re-encode images to ensure malicious code cannot be embedded.
* **Storage Controls:** Store uploaded files outside the web root directory and disable script execution permissions on upload folders.

#### Result
Demonstrated that an attacker can upload arbitrary code to the server web directory, leading to complete remote server compromise.

<img width="813" height="274" alt="fileuplaod-zap03" src="https://github.com/user-attachments/assets/576129a5-5548-4596-a880-a22f63757317" />
<img width="1389" height="865" alt="file-upload" src="https://github.com/user-attachments/assets/4119baac-53fa-47e1-9ec0-ba7d40b289ca" />
<img width="1566" height="58" alt="fileupload04" src="https://github.com/user-attachments/assets/44120ed3-aea2-406e-8b11-8cb06273f832" />
<img width="822" height="243" alt="fileupload-zap" src="https://github.com/user-attachments/assets/5b040942-7ef7-4e94-bbb6-028e44533495" />

---

### Module 6: Insecure CAPTCHA
* **Endpoint:** `/vulnerabilities/captcha/`
* **Status:** Skipped / Pending Configuration Fix

#### Note:
This module could not be tested because the containerized environment restricts direct configuration file edits. Without valid reCAPTCHA API keys defined, the application suppresses the password-reset form inputs entirely, preventing local interaction with the workflow.

---

### Module 7: SQL Injection (Error-Based & Blind)
* **Endpoints:**
        1. Standard: `/vulnerabilities/sqli/`
        2. Blind: `/vulnerabilities/sqli_blind/`
* **Vulnerable Parameter:** `id`
* **Severity:** High

#### Vulnerability Details:
The target application incorporates user-supplied input directly into database queries without using parameterized statements or proper input validation. This allows attackers to manipulate the database logic by appending malicious SQL commands.

#### Testing Steps:
* **Standard SQL Injection Testing:**
1. Navigated to `/vulnerabilities/sqli/`.
2. Injected a boolean-always-true payload (`1' OR '1'='1`) into the `id` input field.
3. Observed the database query evaluation via the browser network tab and application response.

* **Blind SQL Injection Testing:**
1. Navigated to `/vulnerabilities/sqli_blind/`
2. Submitted the same logical payload (`1' OR '1'='1`) into the `id` field.
3. Monitored the application's conditional state responses since data is not directly printed to the screen.

#### OWASP ZAP & Network Telemetry
* **Standard SQLi Request (`GET`):**
Captured requests targeting `/vulnerabilities/sqli/?id=1%27+OR+%271%27%3D%271&Submit=Submit` returning `200 OK`

* **Blind SQLi Request (`GET`):**
Captured requests targeting `/vulnerabilities/sqli_blind/?id=1%27+OR+%271%27%3D%271&Submit=Submit` returning `200 OK`

#### Results
* **Standard SQLi Result:** The injected condition forced the database query to evaluate as `TRUE` for all table rows, dumping all user accounts (`admin`, `Gordon`, `Hack`, `Pablo`, `Bob`) directly to the interface.
* **Blind SQLi Result:** Although the database did not print record values, the application confirmed vulnerability by changing state, displaying the affirmative validation message: *"User ID exists in the database."*

#### Impact
Demonstrates critical database exposure where unauthorized users can bypass authentication logic, read sensitive records, or completely compromise backend data storage.

<img width="1494" height="830" alt="sql-blind" src="https://github.com/user-attachments/assets/4adf763d-8a9c-45f3-91e6-e91dcb3303e9" />
<img width="1490" height="815" alt="sql-blind01" src="https://github.com/user-attachments/assets/d447d1f5-223f-4ccc-8bc0-7a43ed328f19" />
<img width="816" height="282" alt="sqli-blind-zap" src="https://github.com/user-attachments/assets/c008cf1b-5b27-41dc-ae97-49905c734c5c" />
<img width="1489" height="824" alt="sql-inj" src="https://github.com/user-attachments/assets/ce4ace97-28b4-41f3-90bd-47ff536020c0" />
<img width="812" height="281" alt="sqli-zap02" src="https://github.com/user-attachments/assets/93948b48-e6aa-40d3-905d-ffe6cc16da97" />
<img width="1614" height="186" alt="sqli-zap" src="https://github.com/user-attachments/assets/36f3f6a1-55a5-4a49-a5af-c571c5b34175" />

---

### Module 8: Weak Session IDs
​* **Endpoint:** /vulnerabilities/weak_id/
​* **Severity:** Medium

​#### Vulnerability Details:
​The application generates session tokens (cookies) using a predictable pattern or incremental counter (such as sequential numbers). Because the values follow a recognizable sequence, an attacker can easily anticipate or calculate future session IDs to hijack active user accounts.
​
#### Testing Steps:
1. Navigated to the Weak Session IDs endpoint (/vulnerabilities/weak_id/).
​2. Clicked the Generate button multiple times to issue new dvwa Session cookies.
3. Monitored OWASP ZAP and browser tools to inspect the generated session values.

​#### OWASP ZAP:
​* **Proxy History Log (POST):**
​26/10/2026, 07:28:02 to 07:28:08 – Multiple sequential POST requests were captured hitting /vulnerabilities/weak_id/, returning 200 OK status codes for each generated token. 
 
​* **Cookie Inspection:**
​Browser storage telemetry revealed that dvwaSession tokens incremented sequentially (e.g., moving from 17 up through 19 and 20), rather than utilizing a cryptographically secure random number generator (CSPRNG).  

#### Result
Demonstrated that session tokens can be easily anticipated or brute-forced, allowing unauthorized actors to forge valid session tokens and impersonate legitimate authenticated users.

<img width="1495" height="841" alt="weak-session-id" src="https://github.com/user-attachments/assets/f0c47001-10aa-46b2-871c-cf39921aa90a" />
<img width="824" height="336" alt="weak-session-zap" src="https://github.com/user-attachments/assets/572fad70-1e89-4a93-8420-e940b9e1b51a" />
<img width="1626" height="302" alt="weak-session-zap02" src="https://github.com/user-attachments/assets/cc4e9d83-3245-49cc-92eb-20a673e55eb9" />

---

### Module 9: XSS (DOM, Reflected, Stored)
* **Endpoints:**
1. DOM: `/vulnerabilities/xss_d/`
2. Reflected: `/vulnerabilities/xss_r/`
3. Stored: `/vulnerabilities/xss_s/`
* **Severity:** High

#### Vulnerability Details
The application handles user input unsafely across three distinct vectors without proper sanitization or output encoding:
1. Reflected XSS: Immediately echoes input from HTTP query parameters back into the browser's response page.
2. DOM-Based XSS: Client-side scripts process input directly from the browser's URL parameters or environment locally.
3. Stored XSS: Saves malicious script input permanently into the backend database (such as guestbook comments) to execute for every visiting user.

#### Testing:
* **Reflected Testing:** Injected a lowercase script payload (`<script>alert('reflect')</script>`) via the input parameter. 
Network Tab Telemetry: 
The browser inspector captured the incoming `GET` request to `/vulnerabilities/xss_r/` returning a `200 OK` status with the payload fully reflected in the request URI.
* **DOM Testing:** Appended script payloads to the URL parameters (`?default=<script>alert('DOM')</script>`) to manipulate client-side JavaScript execution.
* **Stored Testing:** Posted script payloads into the guestbook message fields to ensure persistence and automatic execution on reload.

#### Result
Successfully demonstrated that arbitrary JavaScript can be executed within the victim's browser context across all three vectors, highlighting risks like session cookie theft, account compromise, and unauthorized actions.

<img width="1395" height="817" alt="xss-dom" src="https://github.com/user-attachments/assets/ab2dd086-7c37-4e42-b667-d282df582786" />
<img width="1393" height="835" alt="xss-reflacted" src="https://github.com/user-attachments/assets/c3e3a800-582c-4c18-897c-661d46b8f686" />
<img width="1394" height="835" alt="xss-storerd" src="https://github.com/user-attachments/assets/85979cfd-26b6-44b4-aad8-ae8f1a2a20cc" />
<img width="1618" height="282" alt="xss-zap" src="https://github.com/user-attachments/assets/1bb4d264-b2f8-4065-9df2-ee0fdce303c0" />

---

### Module 10: CSP Bypass
* **Endpoint:** `/vulnerabilities/csp/`
* **Severity:** Medium

#### Vulnerability Details
Content Security Policy (CSP) restricts the external domains from which a browser is allowed to load and execute scripts. In this module, the policy configuration or input handling permits external domain references, which can be tested using external script endpoints or JSONP callback vectors.

#### Testing Details:
* **Payload Tested:**
Submitted an external script URL parameter targeting an external endpoint (e.g., `https://duckduckgo.com/?q=ABC&callback=bypassalert`).

* **Network Tab Telemetry:**
The browser initiated a `POST` request to `/vulnerabilities/csp/`.
 A subsequent external `GET` request was dispatched to `duckduckgo.com` to fetch the script payload, resulting in policy evaluation.

#### Result
Demonstrates how content security policies interact with external script requests. When weak or permissive rules allow third-party script inclusion endpoints, it highlights the risks of policy bypasses via trusted domain endpoints or JSONP callbacks.

<img width="1395" height="784" alt="csp" src="https://github.com/user-attachments/assets/fd5f540d-a6b8-46c6-a83e-9bee2dc355d1" />
<img width="1591" height="59" alt="csp_zap" src="https://github.com/user-attachments/assets/d70940cb-ada2-40ef-9d09-038a41eeee64" />


---

### Module 11: JavaScript Attacks
* **Endpoint:** `/vulnerabilities/javascript/`
* **Severity:** Low / Medium

#### Vulnerability Details:
This module demonstrates the risks of relying strictly on client-side controls (code running in the user's browser) for security validation. The web page uses a local script to generate a hidden token and check whether a user entered the correct phrase ("success") before allowing a win state. Because these scripts run entirely on the client side, users can inspect, modify, and override them

#### OWASP ZAP 
During testing, OWASP ZAP logged the traffic flow for this module:
* **Page Access:** 
  * `26/10/2026, 05:54:14` – `GET` request to `/vulnerabilities/javascript/` (`200 OK`)
* **Form Submission:** 
  * `26/10/2026, 06:03:40` & `06:07:18` – `POST` requests submitted after interacting with the script controls (`200 OK`)

#### Exploitation Steps
1. Opened the webpage and observed that typing a regular phrase resulted in an "Invalid token" error.
2. Opened the browser's Developer Tools Console (`F12`).
3. Manually forced the input value to the required string and executed the hidden token script directly from the console:
   ```javascript
   document.getElementById("phrase").value = "success";
   generate_token();
   
#### Result: 
Client-side security controls can be completely bypassed using browser dev tools.

<img width="1397" height="923" alt="javascript" src="https://github.com/user-attachments/assets/82621b2c-5399-4b66-b784-a8bddfae616d" />
<img width="1395" height="884" alt="javascript02" src="https://github.com/user-attachments/assets/fd83714c-945b-4a99-8ec6-21df11c59b5f" />
<img width="1639" height="62" alt="javscript_zap" src="https://github.com/user-attachments/assets/dba86727-58f4-466d-ba30-81e85dcb4e89" />

---

### Module 12: Authorisation Bypass
* **Endpoint:** /vulnerabilities/authbypass/
* **Severity:** High / Critical

#### Vulnerability Details:
This module demonstrates broken access control flaws where low-privileged users can access administrative features due to improper server-side authorization checks. While the page is intended strictly for the `admin` account, the application fails to enforce permission validation on request parameters, allowing unauthorized users (e.g., `gordonb` / `abc123`) to view and modify user account details.

#### Exploitation Steps:
1. Intercept HTTP requests using OWASP ZAP configured on proxy port `8081`.
2. Authenticate or navigate to the Authorisation Bypass endpoint (`/vulnerabilities/authbypass/`).
3. Modify user details within the User Manager form fields (such as updating names for entries like `root`, `writeups`, `Auth`).
4. Submit the modified parameters directly to the server without administrative re-authentication.

#### OWASP ZAP:
During manual proxy testing, OWASP ZAP logged the following HTTP activity:

* **Endpoint Access (`GET`):**
  `26/10/2026, 05:20:06` – `GET` `/vulnerabilities/authbypass/` returning status `200 OK`.
* **Unauthorized Modification (`POST`):**
   `26/10/2026, 05:21:28` to `05:21:30` – Sequential `POST` requests sent to `/vulnerabilities/authbypass/ch...`
   The server returned HTTP `200 OK` for each request, confirming profile modification attempts were accepted.

#### Result:
The application displayed a `"Save Successful"` notification and updated user record entries (such as `root`, `writeups`, `Auth`, `Gordon`, `admin`). This confirms that horizontal and vertical access controls were absent, allowing non-admin sessions to perform administrative updates

<img width="1636" height="171" alt="authbypass-zap" src="https://github.com/user-attachments/assets/b0b3606f-41c5-4bba-b017-a7b5e55ddceb" />
<img width="1260" height="870" alt="authbypass" src="https://github.com/user-attachments/assets/1c7bd51d-21ca-4f61-97db-c3583628ec81" />

---

### Module 13: Open HTTP Redirect
* **Endpoint:** `/vulnerabilities/open_redirect/`
* **Severity:** Low / Medium

#### Exploitation Details
The application handles redirect parameters and source info pages (`/vulnerabilities/open_redirect/source/info.php?id=1`) where input handling can be manipulated

#### OWASP ZAP:
During manual proxy exploration, OWASP ZAP recorded the exact HTTP request sequence as parameters were manipulated:

#### Intercepted History Log:
  * `26/10/2026, 05:06:05` – `GET` `/vulnerabilities/open_redirect/`
  * `26/10/2026, 05:06:05` to `05:06:16` (IDs 51–54) – Sequential `GET` requests to `/vulnerabilities/open_redirect/...` showing rapid parameter changes on the local target (`http://127.0.0.1:8080`)

#### Observation During Testing
Manipulating parameters like the `id` field (e.g., changing `id=1` to other values) cycles through internal database records or hacker quote entries. While lower risk for static test content, unvalidated parameter routing or open redirects can become severe vulnerabilities in production environments if abused in phishing campaigns to redirect users to malicious domains, or if combined with object enumeration

#### Result
Demonstrates a lack of strict input sanitization and destination validation on routing parameters. It can be abused in phishing campaigns to trick users into navigating to malicious external domains while trusting the original hostname; however, during direct parameter manipulation, it functions closer to an IDOR (Insecure Direct Object Reference) behavior.

<img width="1636" height="135" alt="open-http-redirect-zap" src="https://github.com/user-attachments/assets/68ee5ceb-8162-4480-aa70-2fee811fd768" />

---

## 3. Conclusion

This hands-on security assessment of the Damn Vulnerable Web Application (DVWA) at the Low security level highlights the fundamental risks associated with unvalidated inputs, insecure state management, broken access controls, and client-side trust dependencies. 

By utilizing **OWASP ZAP** for request interception, parameter manipulation, and fuzzing, this assessment successfully documented vulnerabilities ranging from medium-severity information exposure to critical. Implementing the recommended server-side mitigations—such as input sanitization, strict whitelisting, parameterized queries, and anti-CSRF tokens—is vital for hardening web applications against similar exploitation vectors. 


