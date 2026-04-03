## Introduction to Command Injections

Command injection sits at the intersection of two fundamental web security problems: trusting user input and passing that input to a privileged execution context. The operating system shell has no concept of "intended" versus "injected" input. It processes whatever string it receives, making any unsanitised user data reaching a shell command an immediate and complete security boundary violation.

***

## Injection Vulnerabilities in Context

[OWASP ranks injection as the third most critical web application risk](https://owasp.org/www-project-top-ten/), reflecting both its prevalence and the severity of its consequences. The underlying pattern is consistent across every injection type: user-supplied input crosses a trust boundary and is interpreted as code or a query rather than data.

| Injection Type | Execution Context | Example Vulnerable Function |
|---------------|------------------|---------------------------|
| OS Command | System shell | `system()`, `exec()`, `child_process.exec()` |
| Code Injection | Language interpreter | `eval()`, `assert()` |
| SQL Injection | Database engine | Raw SQL string concatenation |
| XSS/HTML Injection | Victim's browser | Unsanitised output to HTML |
| LDAP Injection | Directory service | Unsanitised LDAP query construction |
| NoSQL Injection | NoSQL database | MongoDB `$where` with user input |
| XPath Injection | XML processor | XPath query construction |

As web technologies evolve and new query types are introduced, new injection categories follow. The root cause remains constant in every case: the application fails to maintain a clear boundary between data and executable code.

***

## OS Command Injection Specifically

OS command injection occurs when user input reaches a function whose job is to invoke the operating system shell. The shell interprets the entire string it receives as a command, and since shell metacharacters like `;`, `&&`, `|`, and `` ` `` have special meaning to the shell but not to the application, they allow an attacker to step outside the intended command and run arbitrary additional ones.

### PHP Execution Functions

PHP provides several functions that invoke system commands, each with slightly different behaviour:

```php
// system(): executes and prints output directly
system("ping -c 4 " . $_GET['host']);

// exec(): executes, returns last line of output
$output = exec("ping -c 4 " . $_GET['host']);

// shell_exec(): executes, returns full output as string
$result = shell_exec("ping -c 4 " . $_GET['host']);

// passthru(): executes and passes raw output to browser
passthru("ping -c 4 " . $_GET['host']);

// popen(): opens a pipe to the process
$handle = popen("ping -c 4 " . $_GET['host'], "r");
```

A concrete vulnerable example: an application that creates PDFs from user-supplied filenames:

```php
<?php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}
?>
```

The developer intended `filename` to be a simple string like `report`. An attacker supplies `report; cat /etc/passwd` and the shell receives:

```bash
touch /tmp/report; cat /etc/passwd.pdf
```

Both commands execute. The `touch` completes normally and the `/etc/passwd` content is returned in the response.

### NodeJS Execution Functions

The same vulnerability pattern appears in NodeJS through the `child_process` module:

```javascript
// child_process.exec(): passes command string to shell
app.get("/createfile", function(req, res) {
    child_process.exec(`touch /tmp/${req.query.filename}.txt`);
});

// child_process.spawn(): takes command and args separately
// This is the SAFER approach as it bypasses the shell
child_process.spawn('touch', ['/tmp/' + req.query.filename + '.txt']);
```

The critical difference between `exec()` and `spawn()` is that `exec()` passes the command to a shell (`/bin/sh -c`), where metacharacters are interpreted. `spawn()` calls `execve()` directly with an argument array, bypassing the shell entirely and making injection impossible regardless of what the filename contains. This is the architectural fix, not input sanitisation.

***

## Why the Impact Is So Severe

Most web vulnerabilities require chaining multiple steps to reach meaningful impact. Command injection collapses this into a single step because the attacker immediately gains:

- **Arbitrary file read:** `cat /etc/passwd`, `cat /var/www/html/config.php`
- **Arbitrary file write:** `echo '...' > /var/www/html/backdoor.php`
- **Network enumeration:** `ifconfig`, `netstat -an`, `ss -tulpn`
- **Credential access:** Config files, environment variables, `.ssh/` directories
- **Reverse shell:** Direct interactive access to the server process

The only remaining step is privilege escalation from the web server user (`www-data`) to `root`, and the attacker already has the tools and access needed to attempt that from their reverse shell session.

***

## The Core Prevention Principle

The fix operates at the architecture level before any sanitisation is considered. The correct order of preference is:

1. **Eliminate shell invocation entirely:** Use language-native libraries. PHP's `socket_connect()` instead of shelling out to `ping`. Python's `zipfile` module instead of calling `zip`. Java's `ProcessBuilder` with explicit argument arrays instead of `Runtime.exec(String)` with a shell string

2. **If shell invocation is unavoidable, use argument arrays:** `spawn()` over `exec()` in NodeJS, `proc_open()` with an array in PHP, `subprocess.run(list)` in Python. These call `execve()` directly and the shell never sees the input

3. **If a shell string is truly necessary, escape the entire argument:** PHP's `escapeshellarg()` wraps the input in single quotes and escapes internal single quotes, neutralising all metacharacters. Python's `shlex.quote()` does the same

4. **Whitelist input validation as a final layer:** If the field accepts only an IP address, reject anything that is not `0-9` and `.`. If it accepts a filename, reject anything outside alphanumeric characters, hyphens, and underscores

Input sanitisation attempting to blacklist dangerous characters like `;`, `&&`, and `|` is the weakest possible mitigation because the list of shell metacharacters is large, encoding bypasses exist, and a single missed character breaks the entire defence.
