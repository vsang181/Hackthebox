# Local File Inclusion: Foundations

File Inclusion vulnerabilities emerge from a fundamental design pattern in web development: using URL parameters to dynamically load content. The vulnerability is not in the pattern itself, but in passing user-controlled values directly into filesystem functions without sanitisation.

***

## Where LFI Typically Hides

Templating engines are the most common location because they are designed to load different content based on a parameter. A URL like `/index.php?page=about` looks completely benign, but if `about` feeds directly into an `include()` call, an attacker substitutes it with a path to any file the web server process can read.  The feature works exactly as intended, which is why it gets missed in code review. [owasp](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/11.1-Testing_for_Local_File_Inclusion)

Common legitimate-looking parameters that are frequently vulnerable:
- `?page=`, `?file=`, `?template=`
- `?lang=` or `?language=`
- `?view=`, `?section=`, `?module=`
- URL path segments like `/about/en/` where `en` controls a directory

***

## Vulnerable Patterns Across Frameworks

The vulnerability has the same root cause in every language: unsanitised user input flowing into a file-loading function.

**PHP:**
```php
// Any of these with user input = vulnerable
include($_GET['language']);
include_once($_GET['language']);
require($_GET['language']);
require_once($_GET['language']);
file_get_contents($_GET['language']);
```

**NodeJS:**
```javascript
// Direct path injection via readFile
fs.readFile(path.join(__dirname, req.query.language), function(err, data) {
    res.write(data);
});

// Path injection via Express render
app.get("/about/:language", function(req, res) {
    res.render(`/${req.params.language}/about.html`);
});
```

**Java (JSP):**
```jsp
<jsp:include file="<%= request.getParameter('language') %>" />
<c:import url="<%= request.getParameter('language') %>"/>
```

**.NET:**
```csharp
Response.WriteFile(HttpContext.Request.Query['language']);
@Html.Partial(HttpContext.Request.Query['language'])
<!--#include file="<% HttpContext.Request.Query['language'] %>"-->
```

Note that the NodeJS `render()` and Express path parameter examples show that the injection point does not always come from a `?query=` GET parameter. It can come from the URL path itself, which is sometimes less expected and easier to overlook.

***

## Read vs Execute: A Critical Distinction

Not all file inclusion functions behave the same way. Whether a function executes the included file or only reads its content determines the maximum severity of exploitation:

| Function | Language | Read | Execute | Remote URL |
|----------|----------|------|---------|------------|
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

Functions that execute files are the most dangerous because they can turn file reading into remote code execution. Functions that only read content are still critical: they expose source code, credentials, keys, and configuration files that can be used to compromise the application through secondary vectors.

Remote URL support is the enabler for Remote File Inclusion (RFI): if the function both executes and accepts remote URLs, an attacker can point it at a server they control hosting malicious code.

***

## Impact Chain

Even "read-only" file inclusion is not a minor issue. The escalation path looks like this:

```
LFI (read-only)
    |
    v
Read /etc/passwd              → Username enumeration
Read application config       → Database credentials, API keys
Read web application source   → Discover other vulnerabilities
Read SSH private keys         → Direct server access
    |
    v
LFI (execute-capable)
    |
    v
Log poisoning                 → Inject PHP into log files, include log
PHP session file inclusion    → Write payload to session, include it
/proc/self/environ injection  → Inject via User-Agent header
    |
    v
Remote Code Execution         → Full back-end server compromise
```

This is why the classification of the function matters when auditing code. `file_get_contents()` on user input is a serious data exposure vulnerability. `include()` on user input is potentially a full server takeover. Both need fixing, but the urgency and the exploitation techniques differ.
