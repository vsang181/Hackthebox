# Intercepting Web Requests

Request interception is the core feature that makes web proxies so powerful. Once a request is intercepted, it hangs in place until you manually forward it, giving you the opportunity to read, modify, and replay it before it ever reaches the back-end server.

***

## Enabling Interception

### Burp Suite

Interception is on by default in Burp. You can toggle it from `Proxy > Intercept` by clicking the "Intercept is on/off" button. When active, any request made through the browser will pause and appear in the Intercept tab waiting for your action. Click "Forward" to send it through, or "Drop" to discard it entirely.

If you see requests from other browser activity appearing before your target request, keep clicking Forward until you reach the one you are interested in.

### ZAP

In ZAP, interception is off by default, indicated by the green button on the top toolbar (green means requests pass through freely). Click the button or press `CTRL+B` to toggle it on. When a request is intercepted it appears in the top-right pane, and you can click the "Step" button (next to the red break button) to forward it.

ZAP also offers the Heads Up Display (HUD), which surfaces proxy controls directly inside the pre-configured browser without needing to switch back to the ZAP interface. Enable it from the button at the end of the top menu bar. The HUD's left pane has an interception toggle button as its second option from the top. When a request is intercepted through the HUD, you are given two choices:

- **Step**: Forward the current request and break on the next one
- **Continue**: Forward all remaining requests without breaking

Step is useful when you want to trace every individual action a page takes, while Continue is the better choice when you only care about one specific request and want everything else to pass through uninterrupted.

***

## Manipulating Intercepted Requests

Once a request is intercepted you can edit any part of it directly in the proxy interface before forwarding. This includes headers, parameter values, cookies, the request method, and the body.

The practical value of this becomes clear with a simple example. Consider a ping function on a web page that sends the following POST request:

```http
POST /ping HTTP/1.1
Host: 94.237.62.138:32306
Content-Type: application/x-www-form-urlencoded

ip=1
```

The front-end JavaScript restricts the input field to numeric characters only, so you cannot type a command directly into the browser. However, that restriction only exists in the browser. Once you intercept the request, you can edit the `ip` parameter to anything you want before the server ever sees it:

```
ip=;ls;
```

If the back-end does not independently validate or sanitise that input, the server will execute the injected command and return the output in the response. This is command injection achieved purely through proxy manipulation, bypassing the front-end restriction entirely.

***

## Why This Matters

Front-end validation is cosmetic. It improves user experience but provides zero security because any intercepting proxy can bypass it instantly. Every security control that actually matters must be implemented on the back-end. Demonstrating this to clients during a pentest, particularly on applications that developers assumed were "protected" by JavaScript input restrictions, is one of the most impactful findings you can surface.

Request interception and manipulation is the foundation for manually testing a wide range of vulnerabilities:

- SQL injection via parameter tampering
- Command injection as shown above
- File upload bypass by changing MIME types or file extensions in the request
- Authentication bypass by modifying session tokens or role values
- XSS payload delivery through headers or body parameters
- XXE via modified XML content
- Insecure deserialisation through manipulated serialised objects
- Error handling analysis by sending malformed or unexpected values

Each of these techniques is covered in depth in dedicated HTB Academy web modules. The proxy is the tool that makes all of them accessible without writing any custom code.
