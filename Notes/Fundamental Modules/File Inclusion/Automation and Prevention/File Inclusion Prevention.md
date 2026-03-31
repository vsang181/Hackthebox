# File Inclusion Prevention

Prevention works in layers. No single control eliminates the risk entirely, but stacking them means an attacker who bypasses one layer immediately encounters the next, and each additional layer generates more detectable noise.

***

## Eliminating the Root Cause

The most effective fix is removing user input from file-loading functions entirely. If the page needs to dynamically load content, the logic should select files internally based on validated identifiers rather than raw user-supplied paths.

The safest pattern is an explicit allowlist map where user input selects an identifier, and the identifier maps to a hardcoded file path:

```php
// Safe: user input never touches the filesystem directly
$pages = [
    'home'    => '/var/www/html/pages/home.php',
    'about'   => '/var/www/html/pages/about.php',
    'contact' => '/var/www/html/pages/contact.php'
];

$page = $_GET['page'];

if (array_key_exists($page, $pages)) {
    include($pages[$page]);
} else {
    include($pages['home']); // safe default
}
```

The attacker can only supply one of the three keys. There is no path in the request at all. This pattern completely eliminates LFI, path traversal, and PHP wrapper attacks in one change because the include function never receives user-controlled data.

For existing large applications where refactoring is not feasible, the same logic applies through a database table that maps IDs to file paths, or a JSON config file that maps slugs to templates. The user supplies a key, the application resolves the path.

***

## Preventing Directory Traversal

Where some degree of path handling is unavoidable, use the language's built-in path resolution functions rather than writing custom sanitisation. Custom sanitisation consistently misses edge cases that native functions handle correctly.

**PHP `basename()`** strips everything except the final filename component, making traversal sequences harmless:

```php
// Strips any directory components, only filename remains
$file = basename($_GET['language']);
include('./languages/' . $file);

// basename('../../../../etc/passwd') returns 'passwd'
// include('./languages/passwd') - file doesn't exist, no traversal
```

The limitation is that `basename()` blocks legitimate subdirectory navigation too. If the application needs to load files from subdirectories, combine it with `realpath()` followed by a strict prefix check:

```php
$base = realpath('./languages/');
$requested = realpath('./languages/' . $_GET['language']);

// realpath() resolves all ../ sequences and symlinks to an absolute path
// If the resolved path doesn't start with the base directory, reject it
if ($requested === false || strpos($requested, $base) !== 0) {
    die('Access denied');
}

include($requested);
```

`realpath()` resolves every `../`, symlink, and encoded character to its true absolute path before the comparison happens, which defeats all traversal bypass techniques discussed in this module.

**Recursive sanitisation** as a secondary control removes `../` sequences even after encoding bypasses produce them:

```php
while (substr_count($input, '../', 0)) {
    $input = str_replace('../', '', $input);
}
```

The `while` loop keeps running until no `../` remains, which defeats the non-recursive filter bypass where `....//` reconstructs `../` after a single pass. This should be used alongside `realpath()` rather than instead of it.

A critical caveat: do not rely on custom sanitisation against shell-executed commands. PHP's `basename()` correctly handles `../../../../etc/passwd`, but if that input reaches a `system()` call in bash, [bash wildcard expansion](https://www.gnu.org/software/bash/manual/bash.html#Filename-Expansion) can treat `.?` and `.*` as traversal equivalents that PHP never sees as dangerous. Native framework functions are safer precisely because their edge cases have been found and fixed by the broader community.

***

## PHP and Web Server Configuration

Server-level controls reduce the impact of any LFI that survives application-level defences.

**php.ini hardening:**

```ini
# Disable all remote file inclusion
allow_url_fopen = Off
allow_url_include = Off

# Restrict PHP file access to the web directory only
open_basedir = /var/www/html

# Disable dangerous extensions
; Remove or comment out:
; extension=expect
```

`open_basedir` enforces a filesystem jail at the PHP engine level. Even if an LFI vulnerability exists and all application-level controls fail, PHP will refuse to open any file outside `/var/www/html`. This directly blocks every sensitive file read demonstrated in this module: `/etc/passwd`, SSH keys, log files, `/proc/` entries, and PHP session files all become inaccessible.

Disabling [PHP Expect](https://www.php.net/manual/en/wrappers.expect.php) removes the `expect://` wrapper attack surface entirely, and disabling [mod_userdir](https://httpd.apache.org/docs/2.4/mod/mod_userdir.html) prevents user home directory access through Apache.

**Container isolation** is the strongest infrastructure-level control. Running the application inside Docker means the container's filesystem contains only the application and its dependencies. Even with a fully working LFI and no `open_basedir`, there are no SSH keys, no `/etc/shadow`, no other users, and no other services to pivot to.

***

## Web Application Firewall

A WAF like [ModSecurity](https://github.com/owasp-modsecurity/ModSecurity) with the [OWASP Core Rule Set](https://coreruleset.org/) detects common traversal sequences, PHP wrapper syntax, and known LFI payloads at the HTTP layer before they reach the application. The most important operational point is starting in permissive (detection-only) mode rather than blocking mode:

```apache
# ModSecurity permissive mode - log but don't block
SecRuleEngine DetectionOnly
```

Permissive mode lets defenders tune rules to eliminate false positives against legitimate traffic before switching to blocking. Even if the organisation never moves to blocking mode, permissive mode functions as an early warning system: a sudden spike in WAF alerts for traversal sequences is often the first sign of an active LFI scan.

The broader point about hardening is that its goal is not to make a system unbreakable but to make attacks louder and slower. The average attacker dwell time before detection was 30 days according to the [FireEye M-Trends 2020 report](https://content.fireeye.com/m-trends/rpt-m-trends-2020). Hardened systems force attackers to leave more artifacts: WAF logs trigger on traversal attempts, `open_basedir` violations appear in PHP error logs, and Docker escape attempts generate unusual syscall patterns. Each layer adds signal that a defender can act on, which is the actual purpose of defence in depth.

***

## Defence Checklist

```
Application layer
    Use allowlist maps: user input selects a key, not a path
    Use realpath() + prefix check for any dynamic path handling
    Use basename() to strip directory components from filenames
    Recursive sanitisation as secondary control only

PHP configuration
    allow_url_fopen = Off
    allow_url_include = Off
    open_basedir = /var/www/html
    Disable expect extension

Infrastructure
    Run application in Docker or equivalent container
    Disable mod_userdir in Apache
    Serve over HTTPS to prevent log injection via cleartext requests

Monitoring
    Deploy ModSecurity in DetectionOnly mode initially
    Tune rules against baseline traffic
    Alert on traversal sequences and PHP wrapper strings in parameters
    Review logs regularly, especially after zero-days for your stack
```
