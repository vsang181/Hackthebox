# Parameter Fuzzing - POST

POST parameter fuzzing follows the same logic as GET fuzzing but targets the request body instead of the URL. The structural difference between the two is small in terms of the ffuf command, but the implications for what you find can be significant.

***

## The Command

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://admin.academy.htb:PORT/admin/admin.php \
     -X POST \
     -d 'FUZZ=key' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs xxx
```

The three additions compared to the GET version:

| Flag | Purpose |
|------|---------|
| `-X POST` | Changes the HTTP method from GET to POST |
| `-d 'FUZZ=key'` | Defines the request body, with `FUZZ` replacing the parameter name |
| `-H 'Content-Type: application/x-www-form-urlencoded'` | Required for PHP to parse the POST body correctly |

Without the `Content-Type` header, a PHP backend will not populate `$_POST` from the request body and will behave as if no parameters were sent at all, making every request look identical and producing no useful results.

***

## Reading the Response

Once the `id` parameter is discovered, verifying it with curl is faster than using a browser for POST requests:

```bash
curl http://admin.academy.htb:PORT/admin/admin.php \
     -X POST \
     -d 'id=key' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

The response `Invalid id!` is a meaningful step forward. It confirms:

- The parameter `id` is actively processed by the backend
- The application has validation logic, meaning it expects a specific format or value
- The error message itself is information: it tells you the parameter name is correct but the value is wrong, rather than ignoring the parameter entirely

This is the transition point from parameter discovery to value fuzzing, which is the next stage.

***

## Why POST Parameters Are Often More Sensitive

POST parameters tend to be associated with actions rather than queries. While GET parameters usually retrieve data, POST parameters typically submit it: login credentials, user IDs, admin actions, file uploads. Finding an undocumented POST parameter on an admin page is a higher-value finding than the equivalent GET parameter, because the backend logic attached to it is more likely to perform privileged operations.

The `id` parameter discovered here is a textbook example. An admin page that accepts an `id` value via POST and returns `Invalid id!` is almost certainly looking up a record by that ID, and the next step is to fuzz for valid ID values.

***

## Value Fuzzing as the Next Step

Now that you have a confirmed parameter name, you shift from fuzzing parameter names to fuzzing parameter values:

```bash
ffuf -w /opt/useful/seclists/Fuzzing/4-digits-0000-9999.txt:FUZZ \
     -u http://admin.academy.htb:PORT/admin/admin.php \
     -X POST \
     -d 'id=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs xxx
```

Here `FUZZ` moves from the parameter name position to the value position. The wordlist changes accordingly: instead of a list of parameter names, you use a numeric range, a list of usernames, UUIDs, or whatever format the application appears to expect. The error message `Invalid id!` suggests a numeric or short alphanumeric ID is expected, making a sequential numeric wordlist the logical first attempt.
