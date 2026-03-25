# Intercepting Responses

While request interception lets you modify what you send to the server, response interception lets you modify what the server sends back before the browser renders it. This is particularly useful for stripping front-end restrictions baked into the HTML itself, such as disabled input fields, hidden elements, or JavaScript-enforced input validation.

***

## Burp Suite

Response interception is not enabled by default in Burp. Enable it by going to `Proxy > Proxy Settings` and turning on "Intercept Response" under the Response Interception Rules section.

Once enabled, do a forced full page refresh with `CTRL+SHIFT+R` in your browser (this ensures the browser requests a fresh copy rather than loading from cache). Burp will first catch the outgoing request; click Forward to pass it through, and then the server's response will be intercepted before it reaches the browser.

From here you can edit the raw HTML directly. Using the ping example from the previous section, you can find the IP input field and change it from:

```html
<input type="number" id="ip" name="ip" min="1" max="255" maxlength="3">
```

to:

```html
<input type="text" id="ip" name="ip" min="1" max="255" maxlength="100">
```

Changing `type="number"` to `type="text"` removes the numeric-only restriction, and extending `maxlength` to `100` gives you enough space to enter a full payload. Click Forward again and the browser renders the modified response, giving you an input field that now accepts any character. You can then type your command injection payload directly in the browser without needing to intercept the request itself.

The same technique works for re-enabling disabled HTML buttons. Find the button element in the intercepted response, remove the `disabled` attribute, forward the response, and the button becomes clickable.

***

## ZAP

ZAP handles this slightly differently. When you use the Step button to forward an intercepted request, ZAP automatically intercepts the corresponding response in the same action. You do not need to enable a separate setting as you do in Burp. Make your changes to the response HTML and click Continue to send it to the browser.

### ZAP HUD Shortcuts

The ZAP HUD provides two particularly useful buttons for response manipulation that do not require intercepting anything at all:

- **Show/Enable button (light bulb icon, third button on the left pane)**: Instantly enables disabled input fields and reveals hidden form fields on the current page without needing to intercept or refresh. This is faster than response interception for simple cases where you just need to interact with a locked element.

- **Comments button**: Displays indicators on the page wherever HTML comments exist in the source. Hovering over an indicator shows the comment content inline, which is useful for finding developer notes, hidden endpoints, credentials, or debug information left in the source code.

Burp has an equivalent to the Show/Enable feature under `Proxy > Proxy Settings > Response Modification Rules`, where you can select options like "Unhide hidden form fields" to apply these changes automatically to every response without manual interception.

***

## When to Use Each Approach

| Scenario | Best Approach |
|---------|--------------|
| Remove input type restriction | Burp/ZAP response interception, edit HTML directly |
| Enable a disabled button quickly | ZAP HUD Show/Enable button |
| Find hidden form fields | ZAP HUD Show/Enable or Burp response modification rules |
| Read HTML comments without viewing source | ZAP HUD Comments button |
| Apply changes to every response automatically | Burp Response Modification Rules (covered in the next section) |

The key takeaway here is that front-end restrictions enforced purely through HTML attributes or JavaScript are always bypassable with a proxy. Any field marked `disabled`, any input type restriction, and any `maxlength` cap can be stripped or modified before the browser ever processes it. If the back-end trusts these restrictions without doing its own validation, the application is vulnerable.
