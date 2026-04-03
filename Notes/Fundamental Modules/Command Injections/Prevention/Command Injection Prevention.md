## Command Injection Prevention

Preventing command injection requires defence in depth across four independent layers: avoiding OS command execution entirely, validating input format, sanitising input content, and hardening the server so that a successful bypass causes minimal damage.

***

## Layer 1: Avoid System Commands Entirely

The most effective prevention is architectural. If your code never calls a shell, shell injection is impossible regardless of what the user submits. Every language provides native library functions that replace the need to shell out for common tasks: 

| Task | Insecure (shells out) | Secure (native library) |
|------|----------------------|------------------------|
| Check host is alive | `system("ping " . $ip)` | `fsockopen($ip, 80)` |
| Send email | `system("sendmail " . $addr)` | `mail()` or PHPMailer |
| Extract archive | `system("unzip " . $file)` | `ZipArchive::open()` |
| Resize image | `system("convert " . $img)` | `Imagick` class |
| DNS lookup | `system("nslookup " . $host)` | `dns_get_record($host)` |
| List directory | `system("ls " . $path)` | `scandir($path)` |

When there is genuinely no native alternative and a system command is required, use an argument array that calls `execve()` directly rather than a shell string. This bypasses the shell parser entirely, making metacharacters irrelevant:

```php
// Insecure: passes through shell, metacharacters interpreted
system("ping -c 1 " . $ip);

// Secure: execve() directly, no shell involved, no injection possible
$args = ['/bin/ping', '-c', '1', $ip];
$proc = proc_open($args, $descriptors, $pipes);
```

```javascript
// Insecure: child_process.exec() uses /bin/sh -c
child_process.exec(`ping -c 1 ${ip}`);

// Secure: child_process.spawn() with array bypasses shell
child_process.spawn('/bin/ping', ['-c', '1', ip]);
```

```python
# Insecure: shell=True passes string to /bin/sh
subprocess.run(f"ping -c 1 {ip}", shell=True)

# Secure: list form calls execve() directly
subprocess.run(['/bin/ping', '-c', '1', ip])
```

***

## Layer 2: Input Validation

Validation checks that input conforms to the expected format and rejects anything that does not. For a field that expects an IP address, literally nothing other than an IP address should pass:

**PHP:**

```php
// Built-in IP validation filter (most reliable)
if (filter_var($_GET['ip'], FILTER_VALIDATE_IP)) {
    // proceed
} else {
    echo "Invalid input";
    die();
}

// Custom format with strict regex (anchored with ^ and $)
if (!preg_match('/^(\d{1,3}\.){3}\d{1,3}$/', $_GET['ip'])) {
    echo "Invalid input";
    die();
}
```

**JavaScript (Node.js):**

```javascript
// Regex validation (anchored to full string)
const ipRegex = /^(25[0-5]|2[0-4][0-9]| [owasp](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection)?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]| [owasp](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection)?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]| [owasp](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection)?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]| [owasp](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection)?[0-9][0-9]?)$/;

if (!ipRegex.test(ip)) {
    res.status(400).send("Invalid input");
    return;
}

// Or use the is-ip library
const isIp = require('is-ip');
if (!isIp(ip)) { return res.status(400).send("Invalid input"); }
```

Validation must be implemented on the back-end. Front-end validation is useful for user experience but provides no security since it can be bypassed in seconds with Burp or curl.

***

## Layer 3: Input Sanitisation

Sanitisation runs after validation and strips any characters that should not exist in the input, even if the format check passed. This provides a second line of defence if the validation regex has an edge case:

**PHP:**

```php
// Whitelist: keep only characters valid for an IP address
$ip = preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);

// For free-text fields where you cannot strip everything,
// escape shell metacharacters (weaker, avoid if possible)
$input = escapeshellarg($_GET['input']); // wraps in single quotes
```

**JavaScript:**

```javascript
// Whitelist: keep only IP-valid characters
var ip = ip.replace(/[^A-Za-z0-9.]/g, '');

// For HTML output contexts, use DOMPurify
import DOMPurify from 'dompurify';
var safeInput = DOMPurify.sanitize(userInput);
```

The critical difference between a blacklist and a whitelist sanitiser:

```
Blacklist approach (weak):
  Remove ;, &, |, $, `, \n from input
  Problem: attacker uses ${IFS}, %0a, character shifting to bypass

Whitelist approach (strong):
  Keep ONLY [A-Za-z0-9.] for an IP field
  Problem for attacker: no bypass exists because the permitted
  character set contains nothing useful for injection
```

A whitelist that only allows alphanumeric characters and a dot for an IP field makes every obfuscation technique covered in this module completely irrelevant. There are no shell metacharacters, no encoding characters, and no expansion operators in `[A-Za-z0-9.]`.

***

## Layer 4: Server Hardening

Even if validation and sanitisation are bypassed, server-level controls limit the blast radius:

**PHP configuration (`php.ini`):**

```ini
# Block all dangerous execution functions
disable_functions = system,exec,shell_exec,passthru,popen,proc_open,
                    pcntl_exec,curl_exec,curl_multi_exec

# Restrict filesystem access to web root only
open_basedir = /var/www/html:/tmp

# Hide PHP version and suppress error output
expose_php = Off
display_errors = Off
log_errors = On
```

**Web server configuration:**

```apache
# Apache: enable ModSecurity WAF
LoadModule security2_module modules/mod_security2.so
SecRuleEngine On

# Run Apache as low-privilege user
User www-data
Group www-data

# Block double-encoded requests
SecRule REQUEST_URI "[\%][0-9a-fA-F]{2}" "deny,status:400"
```

**System-level controls:**

```bash
# Run web application in a container with no network egress
# Prevents reverse shells from phoning home even if RCE is achieved
docker run --network=none --read-only ...

# Use AppArmor or SELinux profile to restrict what the web process can do
aa-enforce /etc/apparmor.d/usr.sbin.apache2
```

***

## Prevention Priority Order

```
1. Replace shell calls with native library functions     (eliminates the vulnerability class)
2. Use argument arrays instead of shell strings          (eliminates shell interpretation)
3. Strict whitelist input validation on back-end         (rejects malformed input)
4. Whitelist-based sanitisation after validation         (strips residual dangerous chars)
5. disable_functions in php.ini                          (limits post-exploitation)
6. open_basedir restriction                              (limits file access after compromise)
7. Least-privilege web server user                       (limits OS access after compromise)
8. WAF as secondary detection layer                      (catches known patterns)
9. Container isolation with no egress                    (limits reverse shell capability)
```

Layers 1 and 2 prevent the vulnerability entirely regardless of what input is supplied. Layers 3 and 4 reduce the exploitable surface. Layers 5 through 9 are damage-control measures for when the upper layers fail, which in a large codebase with many contributors and dependencies, is a realistic scenario that needs to be planned for.
