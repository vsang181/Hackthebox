# Verb Tampering Prevention

Preventing HTTP Verb Tampering requires fixing it at two independent layers: the web server configuration and the application code. A secure server configuration does not protect against insecure code, and secure code does not compensate for a misconfigured server. Both must be addressed.

***

## Fixing Insecure Server Configurations

The fundamental mistake in all vulnerable configurations is the same: access control is applied to a named list of methods, leaving everything else unrestricted. The fix in every web server is to invert the logic and protect everything except what is explicitly permitted.

**Apache (httpd.conf or .htaccess):**

```xml
<!-- Vulnerable: only GET is restricted, POST/HEAD/PUT all bypass auth -->
<Directory "/var/www/html/admin">
    <Limit GET>
        Require valid-user
    </Limit>
</Directory>

<!-- Secure: LimitExcept covers ALL methods except OPTIONS -->
<Directory "/var/www/html/admin">
    AuthType Basic
    AuthName "Admin Panel"
    AuthUserFile /etc/apache2/.htpasswd
    <LimitExcept OPTIONS>
        Require valid-user
    </LimitExcept>
</Directory>
```

`LimitExcept` applies the rule to every method that is NOT listed. Adding any new or arbitrary HTTP method automatically inherits the restriction without requiring a configuration update.

**Tomcat (web.xml):**

```xml
<!-- Vulnerable: only GET is listed in http-method, all others bypass -->
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
        <http-method>GET</http-method>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>

<!-- Secure: http-method-omission covers everything except the listed methods -->
<security-constraint>
    <web-resource-collection>
        <url-pattern>/admin/*</url-pattern>
        <http-method-omission>OPTIONS</http-method-omission>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>
```

**ASP.NET (web.config):**

```xml
<!-- Vulnerable: allow/deny only applies to GET verb -->
<system.web>
    <authorization>
        <allow verbs="GET" roles="admin"/>
        <deny verbs="GET" users="*"/>
    </authorization>
</system.web>

<!-- Secure: remove verb attribute entirely to apply to all methods -->
<system.web>
    <authorization>
        <allow roles="admin"/>
        <deny users="*"/>
    </authorization>
</system.web>
```

Removing the `verbs` attribute in ASP.NET applies the rule to all HTTP methods regardless of which one is used in the request.

***

## Fixing Insecure Code

Server configuration fixes protect the perimeter but do not address logic-level inconsistencies in application code. The vulnerable pattern is always the same: a security check reads from a narrow input source while the execution reads from a broader one.

```php
// Vulnerable: filter reads $_POST, execution reads $_REQUEST
if (isset($_REQUEST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . $_REQUEST['filename']); // uses GET or POST
    } else {
        echo "Malicious Request Denied!";
    }
}
```

The attack path is precise: `$_POST['filename']` is empty on a GET request, so `preg_match` finds nothing to block. `$_REQUEST['filename']` then picks up the GET parameter containing the malicious payload and passes it to `system()`.

```php
// Secure: both filter and execution use the same variable
if (isset($_POST['filename'])) {
    if (!preg_match('/[^A-Za-z0-9. _-]/', $_POST['filename'])) {
        system("touch " . escapeshellarg($_POST['filename']));
    } else {
        echo "Malicious Request Denied!";
    }
}

// Also add explicit method restriction at the top of the function
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    header('Allow: POST');
    die("Method Not Allowed");
}
```

The secure version makes three changes: it checks that the method is POST before doing anything else, reads from `$_POST` in both the filter and the execution, and wraps the shell argument in `escapeshellarg()` as a final layer.

***

## Unified Input Scope Rule

When a function involves security filtering, the scope of the filter must match or exceed the scope of the execution. Using the all-methods parameter superglobal in both places eliminates the method-switching bypass entirely:

| Language | All-Methods Superglobal | Narrow (Vulnerable) |
|----------|------------------------|---------------------|
| PHP | `$_REQUEST['param']` | `$_GET` or `$_POST` alone |
| Java | `request.getParameter('param')` | `request.getQueryString()` alone |
| C# | `Request['param']` | `Request.Form` or `Request.QueryString` alone |

The recommendation is to use the all-methods superglobal in both the security filter and the execution logic, then separately restrict the allowed HTTP method at the entry point of the function. This way the filter covers every possible input path and the method restriction prevents alternative verbs from reaching the function in the first place.

***

## Prevention Checklist

```
Server configuration:
    Use LimitExcept (Apache), http-method-omission (Tomcat),
    or remove verb attribute (ASP.NET) to protect all methods by default
    Never use <Limit> with an explicit list of protected methods
    Disable HEAD requests globally unless specifically required
    Return 405 Method Not Allowed for any unsupported method

Application code:
    Validate the HTTP method at the start of every sensitive function
    Use the same input variable in both the security filter and execution
    Never filter $_POST and execute with $_REQUEST
    Apply escapeshellarg/escapeshellcmd as a final layer on any shell input
    Use static code analysis to flag $_REQUEST in execution after $_POST in filters

Testing:
    Test every security filter with GET, POST, HEAD, PUT, and arbitrary methods
    Confirm that filter bypass via method switch is not possible on any protected endpoint
    Include HTTP verb fuzzing in standard security regression tests
```
