## Preventing File Upload Vulnerabilities

Secure file upload handling requires layered controls that each independently block a different class of attack. No single control is sufficient on its own, but each layer ensures that bypassing one does not immediately lead to compromise.

***

## Extension Validation

The most effective approach combines both a blacklist and a whitelist rather than choosing one over the other. The blacklist catches dangerous extensions that slip through if the whitelist is ever bypassed, while the whitelist ensures only explicitly approved types are accepted:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);

// Blacklist: blocks PHP variants anywhere in the filename
// No $ anchor intentional here - catches shell.php.jpg too
if (preg_match('/^.*\.ph(p|ps|ar|tml)/i', $fileName)) {
    echo "Only images are allowed";
    die();
}

// Whitelist: filename must end with an image extension
// $ anchor is critical - without it shell.jpg.php passes
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/i', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

The `i` flag makes both checks case-insensitive, closing the `shell.PHP` bypass on Linux servers. Both checks use `basename()` on the filename before any validation to strip path components like `../../`, which prevents directory traversal via the filename field. Front-end validation should also be implemented, not because it provides security but because it reduces noise, catches accidental uploads, and means any bypass attempt triggers back-end logging and alerting.

***

## Content Validation

Extension validation alone is insufficient. A file named `shell.jpg` with PHP content and GIF magic bytes bypasses extension checks entirely. Validation must confirm that the extension, the `Content-Type` header, and the actual file MIME type all agree and all match an expected safe type:

```php
$fileName = basename($_FILES["uploadFile"]["name"]);
$contentType = $_FILES['uploadFile']['type'];           // HTTP header (client-supplied)
$MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']); // Actual file content

// Extension must match
if (!preg_match('/^.*\.(png)$/i', $fileName)) {
    echo "Only PNG images are allowed";
    die();
}

// Both Content-Type header and real MIME type must match
foreach (array($contentType, $MIMEtype) as $type) {
    if (!in_array($type, array('image/png'))) {
        echo "Only PNG images are allowed";
        die();
    }
}
```

Checking both `$_FILES['type']` (client-supplied header) and `mime_content_type()` (server-side content inspection) means that spoofing the `Content-Type` header alone is not enough. An attacker would need to correctly forge both the header and embed valid PNG magic bytes, and even then the strict extension whitelist with `$` anchor blocks non-PNG extensions.

For additional assurance on image uploads, running the file through an image validation function confirms it is a parseable image rather than just having the right headers:

```php
// Attempt to get image dimensions - fails on non-images
$imageInfo = getimagesize($_FILES['uploadFile']['tmp_name']);
if ($imageInfo === false) {
    echo "Invalid image file";
    die();
}
```

***

## Concealing the Upload Directory

Direct access to the uploads directory is itself a significant vulnerability regardless of what filtering is in place. Serving files through a controlled download script eliminates the entire class of attacks that require visiting the uploaded file's URL:

```php
// download.php - the only way users can access uploaded files
session_start();

$fileId = intval($_GET['id']);
$userId = $_SESSION['user_id'];

// Fetch the stored filename from database, verify ownership
$stmt = $pdo->prepare("SELECT stored_name, original_name, mime_type 
                        FROM uploads 
                        WHERE id = ? AND user_id = ?");
$stmt->execute([$fileId, $userId]);
$file = $stmt->fetch();

if (!$file) {
    http_response_code(403);
    die("Access denied");
}

$filePath = '/var/uploads/' . $file['stored_name'];

// Validate the resolved path stays within uploads directory
if (strpos(realpath($filePath), realpath('/var/uploads/')) !== 0) {
    http_response_code(403);
    die("Access denied");
}

// Serve with security headers
header('Content-Disposition: attachment; filename="' . $file['original_name'] . '"');
header('Content-Type: ' . $file['mime_type']);
header('X-Content-Type-Options: nosniff');
readfile($filePath);
```

The uploads directory itself should return 403 for all direct requests. In Apache, place a `.htaccess` file in the uploads directory:

```apache
# Deny all direct access to uploads directory
Options -Indexes -ExecCGI
AddHandler default-handler .php .php5 .phtml .phar
Require all denied
```

In Nginx:
```nginx
location /uploads/ {
    deny all;
    return 403;
}
```

### Randomised Storage Names

Storing files with randomised names prevents attackers from predicting or directly accessing uploaded files even if they know the upload directory:

```php
// Generate a random storage name, preserve extension for server handling
$extension = pathinfo($fileName, PATHINFO_EXTENSION);
$storedName = bin2hex(random_bytes(16)) . '.' . $extension;

// Store original name in database, save file with random name
$stmt = $pdo->prepare("INSERT INTO uploads (user_id, original_name, stored_name, mime_type) 
                        VALUES (?, ?, ?, ?)");
$stmt->execute([$userId, $fileName, $storedName, $MIMEtype]);

move_uploaded_file($_FILES['uploadFile']['tmp_name'], '/var/uploads/' . $storedName);
```

This also neutralises filename injection attacks since the original filename never touches the filesystem.

***

## PHP Runtime Hardening

Even if every upload control is bypassed, server-level controls limit what a successfully uploaded and executed webshell can do:

**php.ini dangerous function restrictions:**

```ini
; Disable all functions that execute OS commands
disable_functions = exec,passthru,shell_exec,system,proc_open,popen,
                    curl_exec,curl_multi_exec,parse_ini_file,show_source,
                    pcntl_exec,proc_get_status,proc_nice,proc_terminate

; Restrict filesystem access to web root only
open_basedir = /var/www/html:/tmp

; Hide PHP version and error details from responses
expose_php = Off
display_errors = Off
log_errors = On
error_log = /var/log/php_errors.log
```

Disabling `system`, `exec`, `shell_exec`, and `passthru` means a PHP webshell that relies on these functions cannot execute OS commands even if it successfully uploads and executes. The attacker would need to find alternative execution paths, which are far less common and harder to exploit.

***

## Complete Defence Checklist

```
Extension validation
    Blacklist: blocks PHP variants anywhere in filename (case-insensitive)
    Whitelist: filename must end with approved extension ($-anchored, case-insensitive)
    Apply basename() before any validation to strip path traversal attempts

Content validation
    Validate Content-Type header against allowed MIME types
    Validate actual file MIME type via mime_content_type()
    For images, validate parsability with getimagesize()
    Confirm extension, header, and MIME type all agree

Upload directory protection
    Block all direct access to uploads directory (403)
    Serve files only through a controlled download.php script
    Enforce ownership checks in download script (prevent IDOR)
    Use realpath() + prefix check to prevent LFI via download script
    Store files with random names, keep original name in database

Server hardening
    disable_functions: exec, system, shell_exec, passthru, popen, proc_open
    open_basedir: restrict to /var/www/html and /tmp only
    Store uploaded files on a separate server or container
    Run application in Docker to limit post-exploitation impact

Monitoring and maintenance
    Set maximum file size limit
    Scan uploaded files with ClamAV or equivalent
    Deploy WAF (ModSecurity + OWASP CRS) in detection mode first
    Disable display_errors, log errors server-side only
    Keep all image/file processing libraries updated
    Review WAF logs regularly for upload bypass attempts
```
