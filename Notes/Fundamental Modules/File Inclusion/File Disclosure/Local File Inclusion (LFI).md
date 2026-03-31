# Local File Inclusion: Exploitation Fundamentals

LFI exploitation starts simple and escalates based on how the application handles your input. The core goal in every case is the same: manipulate the file path being passed to the include function to load a file you choose rather than the one the developer intended.

***

## Basic LFI

When an application uses a parameter like `?language=es.php` to load page content, it is almost certainly passing that value directly into an include function. The simplest possible test is to replace it with an absolute path to a known file:

```
/index.php?language=/etc/passwd        # Linux
/index.php?language=C:\Windows\boot.ini  # Windows
```

If the page returns file contents, the vulnerability is confirmed. This works when the backend code uses the parameter with no modification:

```php
include($_GET['language']);
```

The [/etc/passwd file](https://man7.org/linux/man-pages/man5/passwd.5.html) is the standard first target because it exists on every Linux system, requires no elevated permissions to read, and immediately reveals system usernames that can feed into further attacks.

***

## Path Traversal

Most applications do not use the raw parameter value. They prepend a directory path to it:

```php
include("./languages/" . $_GET['language']);
```

Submitting `/etc/passwd` now produces the path `./languages//etc/passwd`, which does not exist. The fix is directory traversal using `../` sequences, where each `../` steps one level up in the filesystem:

```
/index.php?language=../../../../etc/passwd
```

This works because `../` is a standard filesystem navigation sequence that the OS resolves before the file is opened. If you are inside `/var/www/html/languages/`, three levels of `../` brings you to the root. Adding extras beyond the root is harmless since `/../` from `/` still resolves to `/`, so padding with extra `../` sequences never breaks the path and is useful when the application depth is unknown.

To calculate the minimum needed: count the directory depth of the web root. `/var/www/html/` is three levels deep, so `../../../` is the minimum required. Using more than necessary still works but clutters the payload.

***

## Filename Prefix

A slightly different pattern appends the parameter to a string prefix rather than a directory path:

```php
include("lang_" . $_GET['language']);
```

Here, `../../../etc/passwd` becomes `lang_../../../etc/passwd`, which is invalid. The workaround is to prefix your payload with a `/`, forcing the application to treat `lang_` as a directory name rather than part of the filename:

```
/index.php?language=/../../../etc/passwd
```

The resulting path `lang_/../../../etc/passwd` resolves the `/../` to step back past `lang_` into the parent directory, then traverses from there. This does not always succeed since `lang_` as a directory may not exist, and this technique can also interfere with more advanced exploitation methods like [PHP stream wrappers](https://www.php.net/manual/en/wrappers.php) covered in later sections.

***

## Appended Extensions

A common developer mitigation is to append a file extension automatically so the parameter only needs to supply a name:

```php
include($_GET['language'] . ".php");
```

Attempting `/etc/passwd` now requests `/etc/passwd.php`, which does not exist. Older PHP versions (below 5.4) were vulnerable to [null byte injection](https://owasp.org/www-community/attacks/Null_Byte_Injection), where appending `%00` to the path would terminate the string before the `.php` was concatenated:

```
/index.php?language=../../../../etc/passwd%00
```

This is patched in all modern PHP versions. Current bypass techniques for appended extensions involve PHP wrappers and truncation methods, which are covered in later sections of this module.

***

## Second-Order Attacks

Second-order LFI is the most overlooked variant because the injection and the execution are separated by time and application function. The attack works in two stages:

**Stage 1 - Poisoning:** Store a malicious path as a value in a field the application trusts, such as a username during registration:
```
Username: ../../../etc/passwd
```

**Stage 2 - Trigger:** A separate application feature later uses that stored value in a file operation, for example when loading a user avatar:
```
/profile/../../../etc/passwd/avatar.png
```

The application pulls what it thinks is an avatar but is actually reading a system file. Developers miss this because they typically sanitise direct user input from GET/POST parameters but implicitly trust values retrieved from the database, assuming the database only holds clean data.

Second-order vulnerabilities require mapping the full data flow of the application. Any feature that reads a file based on a stored user-controlled value is a candidate: profile pictures, user-specific configuration files, export templates, or any filename stored during registration or account setup.

***

## High-Value Target Files

Regardless of which technique achieves the read, these files are consistently worth targeting on Linux back-end servers:

```
/etc/passwd                          # User accounts and home directories
/etc/shadow                          # Password hashes (requires root access)
/etc/hosts                           # Internal network hostnames
/proc/self/environ                   # Environment variables including secrets
/var/log/apache2/access.log          # Apache access log (used for log poisoning)
/var/log/nginx/access.log            # Nginx equivalent
/var/www/html/config.php             # Application database credentials
/home/user/.ssh/id_rsa               # SSH private key for direct server access
/var/lib/php/sessions/sess_[id]      # PHP session files
```

On Windows targets, the equivalents include `C:\Windows\System32\drivers\etc\hosts`, `C:\xampp\apache\logs\access.log`, and `C:\xampp\htdocs\config.php` depending on the stack in use.
