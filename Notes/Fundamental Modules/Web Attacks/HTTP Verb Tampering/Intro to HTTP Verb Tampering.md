# HTTP Verb Tampering

HTTP Verb Tampering exploits the gap between which HTTP methods a server accepts and which ones it actually applies security controls to. Most developers think exclusively in terms of GET and POST, but the HTTP specification defines nine methods, and web servers often accept several of them without applying the same authentication or filtering logic that protects the standard methods.

***

## The Full HTTP Method Set

Beyond GET and POST, these methods carry significant security implications when left unrestricted:

| Method | Function | Security Risk if Unrestricted |
|--------|----------|------------------------------|
| HEAD | Same as GET but returns headers only, no body | Bypasses auth checks that only cover GET |
| PUT | Writes request payload to specified path | File upload directly to webroot |
| DELETE | Removes resource at specified path | File deletion on server |
| OPTIONS | Returns list of accepted methods | Reveals attack surface |
| PATCH | Applies partial modification to resource | Unauthorised data modification |
| TRACE | Echoes request back to sender | Cross-site tracing (XST) attacks |

OPTIONS is particularly useful during reconnaissance because a request to any endpoint returns an `Allow` header listing every method the server accepts, revealing the full attack surface before any exploitation begins.

***

## Root Cause 1: Insecure Server Configuration

The most direct form of verb tampering arises when server-level authentication configurations explicitly name only certain HTTP methods in their access control rules. The Apache configuration format makes this error easy to introduce:

```xml
<!-- Vulnerable: only GET and POST require authentication -->
<Limit GET POST>
    Require valid-user
</Limit>

<!-- Also vulnerable: explicitly listing methods to require auth leaves HEAD, PUT, DELETE open -->
<LimitExcept GET POST>
    Require valid-user
</LimitExcept>

<!-- Secure: require authentication for ALL methods -->
<LimitExcept OPTIONS>
    Require valid-user
</LimitExcept>
```

The `<Limit>` directive in Apache applies the enclosed rules only to the listed methods. Anything not listed, HEAD, PUT, DELETE, PATCH, or even an invented verb like `CUSTOM`, bypasses the authentication requirement entirely. An attacker sending a HEAD request to a page that requires authentication via `<Limit GET POST>` receives a valid response with no credentials.

The same pattern appears in Tomcat's `web.xml`:

```xml
<!-- Vulnerable Tomcat configuration -->
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
        <http-method>GET</http-method>
        <http-method>POST</http-method>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```

Listing specific `<http-method>` elements means only those methods are restricted. All others are unrestricted. Removing the `<http-method>` elements entirely applies the constraint to all methods, which is the secure configuration.

***

## Root Cause 2: Insecure Coding (Inconsistent Verb Usage)

The second, more common form occurs when a developer applies a security filter to one HTTP method's parameter superglobal but executes the query using a different one. In PHP, `$_GET`, `$_POST`, and `$_REQUEST` are separate:

```php
// Developer intends to sanitise input before SQL query
$pattern = "/^[A-Za-z\s]+$/";

// Filter only checks $_GET['code']
if (preg_match($pattern, $_GET["code"])) {

    // But query uses $_REQUEST['code'], which includes POST parameters too
    $query = "Select * from ports where port_code like '%" 
             . $_REQUEST["code"] . "%'";
}
```

The inconsistency is the attack vector:

```
Normal GET request:
  GET /search.php?code=NORMAL
  $_GET['code'] = "NORMAL" → passes filter
  $_REQUEST['code'] = "NORMAL" → safe query

Attack via POST:
  POST /search.php?code=SAFE
  Body: code=' OR '1'='1

  $_GET['code'] = "SAFE" → passes filter (GET param is safe)
  $_REQUEST['code'] = "' OR '1'='1" → SQL injection executes
```

`$_REQUEST` in PHP combines GET, POST, and cookie parameters. When the filter checks `$_GET` but the query uses `$_REQUEST`, submitting a safe GET parameter that passes the filter while injecting malicious data via POST completely bypasses the sanitisation.

The same inconsistency appears across frameworks:

```javascript
// Node.js/Express equivalent
if (req.query.code.match(/^[A-Za-z]+$/)) {      // checks GET param
    db.query(`SELECT * WHERE code='${req.body.code}'`); // uses POST param
}
```

```python
# Flask equivalent
if re.match(r'^[A-Za-z]+$', request.args.get('code')):  # checks GET
    db.execute(f"SELECT * WHERE code='{request.form.get('code')}'")  # uses POST
```

***

## Impact

Both vulnerability types lead to serious outcomes depending on what the bypassed control was protecting:

- Authentication bypass on admin panels or restricted pages
- Security filter bypass enabling SQL injection, XSS, or command injection on otherwise-protected inputs
- Unauthorised file operations via PUT or DELETE on misconfigured WebDAV-enabled servers
- Access to sensitive functionality hidden behind verb-specific access controls

The insecure coding variant is more prevalent in real applications because it is a subtle logical error that passes code review, whereas the server configuration variant is more visible and explicitly warned against in Apache and Tomcat documentation. Both produce equally exploitable results despite their different origins.
