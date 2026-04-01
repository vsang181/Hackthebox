## File Upload Attacks

File upload functionality is one of the most commonly exploited features in web applications because it sits at the intersection of user convenience and server-side trust. Every upload form is an invitation for a user to place a file on the server, and the security of that interaction depends entirely on how well the application validates what is being placed there.

***

## Why Upload Vulnerabilities Are So Dangerous

Most web vulnerabilities require multiple steps to reach meaningful impact. File upload vulnerabilities can collapse the entire attack chain into a single action: upload a webshell, receive remote code execution. This is why [CWE-434 (Unrestricted Upload of File with Dangerous Type)](https://www.cvedetails.com/vulnerability-list/cweid-434/vulnerabilities.html) consistently produces High and Critical severity CVEs across virtually every category of web application.

The severity scales with what the upload allows:

- **Unauthenticated arbitrary upload**: Any visitor can upload any file type. This is one request away from full server compromise and represents the worst case
- **Authenticated arbitrary upload**: Requires an account but places no restriction on file type. Still critical because accounts are cheap to create or steal
- **Restricted type upload with weak validation**: Only certain types allowed, but the validation is bypassable. High severity depending on bypass complexity
- **Restricted type upload with strong validation**: Genuinely limited to safe file types with multiple validation layers. Low to no upload-specific risk, though other attack paths may still exist

***

## The Primary Attack: Web Shell Upload

The most direct impact from a file upload vulnerability is uploading a web shell, a server-side script that accepts commands through HTTP requests and executes them on the server. A minimal PHP webshell is a single line:

```php
<?php system($_GET['cmd']); ?>
```

Once uploaded and accessible via URL, every HTTP request to that file is a command execution:

```
http://target.com/uploads/shell.php?cmd=id
http://target.com/uploads/shell.php?cmd=cat+/etc/passwd
http://target.com/uploads/shell.php?cmd=which+python3
```

From there, upgrading to a full reverse shell is a single command. The web server process user (`www-data`, `apache`, `nginx`) becomes the attacker's foothold, and post-exploitation begins from that position.

***

## Secondary Attacks When Arbitrary Upload Is Blocked

When the application restricts file types and the restriction holds, the upload functionality can still be exploited for secondary attacks depending on how uploaded content is handled:

**Stored XSS via SVG or HTML upload:** If the application allows SVG files and renders them in the browser, an SVG containing JavaScript executes in the victim's browser context:

```xml
<svg xmlns="http://www.w3.org/2000/svg">
  <script>alert(document.cookie)</script>
</svg>
```

**XXE via SVG or Office documents:** SVG and XML-based formats can reference external entities, enabling server-side request forgery or internal file reads through the XML parser:

```xml
<!DOCTYPE test [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<svg>&xxe;</svg>
```

**DoS via decompression bombs:** Uploading a ZIP file that expands to terabytes of data when extracted can exhaust disk space or CPU, taking down the server.

**Path traversal in filename:** If the application uses the user-supplied filename when saving the file without sanitisation, a filename like `../../var/www/html/shell.php` can place a webshell outside the uploads directory.

**Overwriting critical files:** The same filename traversal can target existing application files. Uploading a malicious `config.php` or `.htaccess` to overwrite the legitimate version changes application behaviour for all users.

***

## Root Causes

Upload vulnerabilities generally stem from one of three sources:

**Weak validation logic** built by developers who do not account for all bypass vectors. Common weak patterns include checking only the file extension (bypassable with double extensions like `shell.php.jpg`), checking only the MIME type from the `Content-Type` header (bypassable by modifying the request), or checking magic bytes without validating the full file content.

**Missing validation entirely**, which is more common than it sounds in internal tools, admin panels, and legacy applications that predate modern security awareness.

**Outdated libraries** with known vulnerabilities in their file processing logic. Image processing libraries, PDF parsers, and archive handlers have historically been rich sources of CVEs that allow code execution through malformed files, entirely separate from whether the application intended to restrict uploads.

***

## What This Module Covers

The following sections work through the full attack and bypass methodology:

```
Basic upload → webshell execution
    |
    v
Client-side filter bypass (JS validation)
    |
    v
Blacklist bypass (double extensions, case variation)
    |
    v
Whitelist bypass (content-type spoofing, magic bytes)
    |
    v
Type confusion attacks (polyglot files)
    |
    v
Limited upload exploitation (XSS, XXE, DoS)
    |
    v
Prevention and secure coding practices
```

Each bypass technique builds on the previous, reflecting how real upload filters are layered and how attackers work through each layer systematically until one fails.
