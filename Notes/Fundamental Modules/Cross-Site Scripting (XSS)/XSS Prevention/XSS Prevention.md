# XSS Prevention

Preventing XSS requires defence at every layer where user data flows: from the moment it enters the application, through how it is stored, and finally how it is rendered back to users. No single layer is sufficient on its own.

***

## Front-End Defences

Front-end validation is the first gate, but it must never be the only one. It improves user experience and reduces noise, but any attacker can bypass it by sending raw HTTP requests directly, completely skipping the browser.

**Input validation** restricts what formats are accepted before submission:
```javascript
// Reject non-email formats before they reach the server
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test(email);
}
```

**Input sanitisation** using DOMPurify strips dangerous content from whatever does get submitted:
```javascript
import DOMPurify from 'dompurify';
let clean = DOMPurify.sanitize(dirtyInput);
```

### Dangerous Sinks to Avoid in Front-End Code

Never write raw user input into any of these:

| Avoid (raw HTML write) | Use Instead |
|------------------------|-------------|
| `element.innerHTML = userInput` | `element.textContent = userInput` |
| `document.write(userInput)` | DOM creation via `createElement` |
| `$.html(userInput)` | `$.text(userInput)` |
| `$.append(userInput)` | Sanitise first with DOMPurify |
| `element.outerHTML = userInput` | Rebuild element safely |

The rule is simple: `textContent` and `innerText` treat input as literal text and never parse it as HTML. `innerHTML` and all equivalent functions parse whatever they receive, including script tags and event handlers.

User input should never be placed directly inside `<script>` blocks, `<style>` blocks, HTML attribute values, or HTML comments, regardless of any sanitisation applied beforehand.

***

## Back-End Defences

Back-end sanitisation is the real line of defence because it cannot be bypassed by the attacker the way front-end controls can.

**Input validation** on the back-end mirrors the front-end but is authoritative:
```php
// PHP: reject anything that is not a valid email
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // proceed
} else {
    // reject - do not store or display
}
```

**Input sanitisation** escapes dangerous characters before storage:
```php
// PHP: escape special characters with backslash
$clean = addslashes($_GET['email']);
```

**Output encoding** is the most critical back-end control. It converts special characters into their HTML entity equivalents so they display correctly but cannot execute:
```php
// PHP: encode on output, not just on input
echo htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8');
echo htmlentities($userInput, ENT_QUOTES, 'UTF-8');
```

```javascript
// NodeJS equivalent
import encode from 'html-entities';
encode('<script>'); // renders as &lt;script&gt; in the browser
```

The distinction between sanitisation (removing dangerous content) and encoding (rendering it harmless as display text) matters. Sanitisation removes `<script>` entirely. Encoding turns `<script>` into `&lt;script&gt;` so it displays as literal text. Encoding is preferred when you need to display the original input back to users.

***

## Server and Infrastructure Configuration

Beyond application code, server-level controls add a layer of protection that operates independently of whether developers wrote secure code:
## XSS Prevention

Preventing XSS requires defence at every layer where user data flows: from the moment it enters the application, through how it is stored, and finally how it is rendered back to users. No single layer is sufficient on its own.

***

## Front-End Defences

Front-end validation is the first gate, but it must never be the only one. It improves user experience and reduces noise, but any attacker can bypass it by sending raw HTTP requests directly, completely skipping the browser.

**Input validation** restricts what formats are accepted before submission:
```javascript
// Reject non-email formats before they reach the server
function validateEmail(email) {
    const re = /^(([^<>()[\]\\.,;:\s@\"]+(\.[^<>()[\]\\.,;:\s@\"]+)*)|(\".+\"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/;
    return re.test(email);
}
```

**Input sanitisation** using DOMPurify strips dangerous content from whatever does get submitted:
```javascript
import DOMPurify from 'dompurify';
let clean = DOMPurify.sanitize(dirtyInput);
```

### Dangerous Sinks to Avoid in Front-End Code

Never write raw user input into any of these:

| Avoid (raw HTML write) | Use Instead |
|------------------------|-------------|
| `element.innerHTML = userInput` | `element.textContent = userInput` |
| `document.write(userInput)` | DOM creation via `createElement` |
| `$.html(userInput)` | `$.text(userInput)` |
| `$.append(userInput)` | Sanitise first with DOMPurify |
| `element.outerHTML = userInput` | Rebuild element safely |

The rule is simple: `textContent` and `innerText` treat input as literal text and never parse it as HTML. `innerHTML` and all equivalent functions parse whatever they receive, including script tags and event handlers.

User input should never be placed directly inside `<script>` blocks, `<style>` blocks, HTML attribute values, or HTML comments, regardless of any sanitisation applied beforehand.

***

## Back-End Defences

Back-end sanitisation is the real line of defence because it cannot be bypassed by the attacker the way front-end controls can.

**Input validation** on the back-end mirrors the front-end but is authoritative:
```php
// PHP: reject anything that is not a valid email
if (filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)) {
    // proceed
} else {
    // reject - do not store or display
}
```

**Input sanitisation** escapes dangerous characters before storage:
```php
// PHP: escape special characters with backslash
$clean = addslashes($_GET['email']);
```

**Output encoding** is the most critical back-end control. It converts special characters into their HTML entity equivalents so they display correctly but cannot execute:
```php
// PHP: encode on output, not just on input
echo htmlspecialchars($userInput, ENT_QUOTES, 'UTF-8');
echo htmlentities($userInput, ENT_QUOTES, 'UTF-8');
```

```javascript
// NodeJS equivalent
import encode from 'html-entities';
encode('<script>'); // renders as &lt;script&gt; in the browser
```

The distinction between sanitisation (removing dangerous content) and encoding (rendering it harmless as display text) matters. Sanitisation removes `<script>` entirely. Encoding turns `<script>` into `&lt;script&gt;` so it displays as literal text. Encoding is preferred when you need to display the original input back to users.

***

## Server and Infrastructure Configuration

Beyond application code, server-level controls add a layer of protection that operates independently of whether developers wrote secure code:

**HTTP security headers:**
```
# Restrict script execution to same origin only
Content-Security-Policy: script-src 'self'

# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Block reflected XSS in older browsers
X-XSS-Protection: 1; mode=block
```

**Cookie flags** are particularly important since session hijacking is the most damaging XSS consequence:
```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

- `HttpOnly` blocks JavaScript from reading the cookie via `document.cookie`, directly neutralising the session hijacking attack covered in the previous section
- `Secure` ensures the cookie is only transmitted over HTTPS, preventing interception
- `SameSite=Strict` blocks the cookie from being sent with cross-site requests

**HTTPS across the entire domain** prevents network-level interception of both session cookies and injected payloads in transit.

***

## Defence in Depth: Layered Model

```
User Input
    |
    v
[Front-end validation]     ← Format checks, type restrictions
    |
    v
[Back-end validation]      ← Authoritative format enforcement
    |
    v
[Back-end sanitisation]    ← Strip or escape dangerous characters
    |
    v
[Database storage]
    |
    v
[Output encoding]          ← htmlspecialchars / html-entities on render
    |
    v
[CSP headers]              ← Block execution even if encoding fails
    |
    v
[HttpOnly cookies]         ← Limit damage if execution occurs
    |
    v
[WAF]                      ← Network-level detection and blocking
```

Each layer compensates for failures in the layers above it. A WAF alone is bypassable. Output encoding alone does not help if a developer later passes encoded output through `innerHTML`. The combination of all layers makes a successful end-to-end XSS attack significantly harder to achieve in practice.

The most common real-world gap is developers who sanitise on input but forget to encode on output, or who apply encoding in most places but miss a single admin-facing template that displays raw user data, which is exactly the blind XSS scenario covered in the session hijacking section.
**HTTP security headers:**
```
# Restrict script execution to same origin only
Content-Security-Policy: script-src 'self'

# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Block reflected XSS in older browsers
X-XSS-Protection: 1; mode=block
```

**Cookie flags** are particularly important since session hijacking is the most damaging XSS consequence:
```
Set-Cookie: session=abc123; HttpOnly; Secure; SameSite=Strict
```

- `HttpOnly` blocks JavaScript from reading the cookie via `document.cookie`, directly neutralising the session hijacking attack covered in the previous section
- `Secure` ensures the cookie is only transmitted over HTTPS, preventing interception
- `SameSite=Strict` blocks the cookie from being sent with cross-site requests

**HTTPS across the entire domain** prevents network-level interception of both session cookies and injected payloads in transit.

***

## Defence in Depth: Layered Model

```
User Input
    |
    v
[Front-end validation]     ← Format checks, type restrictions
    |
    v
[Back-end validation]      ← Authoritative format enforcement
    |
    v
[Back-end sanitisation]    ← Strip or escape dangerous characters
    |
    v
[Database storage]
    |
    v
[Output encoding]          ← htmlspecialchars / html-entities on render
    |
    v
[CSP headers]              ← Block execution even if encoding fails
    |
    v
[HttpOnly cookies]         ← Limit damage if execution occurs
    |
    v
[WAF]                      ← Network-level detection and blocking
```

Each layer compensates for failures in the layers above it. A WAF alone is bypassable. Output encoding alone does not help if a developer later passes encoded output through `innerHTML`. The combination of all layers makes a successful end-to-end XSS attack significantly harder to achieve in practice.

The most common real-world gap is developers who sanitise on input but forget to encode on output, or who apply encoding in most places but miss a single admin-facing template that displays raw user data, which is exactly the blind XSS scenario covered in the session hijacking section.
