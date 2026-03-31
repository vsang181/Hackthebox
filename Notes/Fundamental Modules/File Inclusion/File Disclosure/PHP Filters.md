# PHP Filters for Source Code Disclosure

PHP wrappers extend what LFI can do beyond simply reading flat files. When a target application is PHP-based, the [php:// stream wrapper](https://www.php.net/manual/en/wrappers.php.php) opens up a separate class of attacks that operate entirely within PHP's own I/O layer, without needing to touch the filesystem in the traditional sense.

***

## What PHP Wrappers Are

[PHP stream wrappers](https://www.php.net/manual/en/wrappers.php) are URI-scheme prefixes that redirect I/O operations through PHP's internal stream handling instead of the standard filesystem. When you pass `php://filter/...` to an `include()` call, PHP intercepts it and processes it as an internal stream rather than a file path. This is why filters work even in contexts where direct path traversal is blocked, and why they are effective for bypassing appended extension restrictions since the wrapper syntax is processed before extension concatenation takes effect in some configurations.

The four available filter categories are [String Filters](https://www.php.net/manual/en/filters.string.php), [Conversion Filters](https://www.php.net/manual/en/filters.convert.php), [Compression Filters](https://www.php.net/manual/en/filters.compression.php), and [Encryption Filters](https://www.php.net/manual/en/filters.encryption.php). For LFI source code disclosure the relevant one is `convert.base64-encode` from the Conversion Filters category.

***

## Why Base64 Encoding Is Needed

When you include a PHP file through a standard LFI, the server executes it rather than returning its source. A file like `config.php` that only sets variables and returns no output produces a blank response. You have included it successfully but learned nothing from it.

The `php://filter` wrapper with `convert.base64-encode` intercepts the file before execution and encodes its raw bytes as base64. The PHP engine receives a base64 string rather than PHP code, so there is nothing to execute and the encoded source is returned directly in the response. This makes it the primary technique for source code disclosure when:

- The target file is PHP and would otherwise execute silently
- The application appends `.php` to the parameter, restricting you to PHP files only
- You need to extract credentials, database keys, or logic from application source files

***

## Step 1: Discover PHP Files

Before reading source code, enumerate what PHP files exist on the server using [ffuf](https://github.com/ffuf/ffuf) against the web root:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://TARGET/FUZZ.php \
     -mc 200,301,302,403
```

The `-mc` flag is important here. Unlike normal browsing, you should include 301, 302, and 403 responses in your scan because you can read the source of restricted or redirect-only pages through the filter wrapper even if you cannot access them normally. A 403 page may contain hardcoded credentials or internal logic that was assumed to be inaccessible.

After reading initial files, scan their source for `include`, `require`, or `file_get_contents` calls referencing other PHP files and repeat the process. Starting from `index.php` and following references outward is an alternative approach that often surfaces files that wordlist fuzzing misses.

***

## Step 2: Construct the Filter Payload

The `php://filter` syntax follows this structure:

```
php://filter/read=<filter_name>/resource=<target_file>
```

For base64 source disclosure:

```
php://filter/read=convert.base64-encode/resource=config
```

Note that the `.php` extension is deliberately omitted from `resource=config` when the application automatically appends `.php`. The wrapper resolves `config` and then PHP appends `.php`, resulting in `config.php` being read. If the application does not append an extension, specify the full filename including extension.

Full request:
```
http://TARGET/index.php?language=php://filter/read=convert.base64-encode/resource=config
```

The response will contain a base64 blob in place of the normal page content where the LFI output would appear.

***

## Step 3: Decode the Output

Copy the full base64 string from the page source rather than the rendered page, as long strings are sometimes truncated in browser rendering. Then decode locally:

```bash
echo 'PD9waHAK...SNIP...KICB9Ciov' | base64 -d
```

Or save directly to a file for easier review:

```bash
echo 'PD9waHAK...SNIP...' | base64 -d > config.php
cat config.php
```

A typical `config.php` will contain database connection details:

```php
<?php
$db_host = "localhost";
$db_user = "root";
$db_pass = "SuperSecretPassword123";
$db_name = "app_database";
$conn = new mysqli($db_host, $db_user, $db_pass, $db_name);
?>
```

***

## Chaining Filters

The `php://filter` wrapper supports chaining multiple filters on a single resource, applied left to right. This is useful when a WAF blocks base64 output detection, or when you need to decompress a file before reading it:

```
# Chain: rot13 then base64 (useful for WAF bypass)
php://filter/read=string.rot13|convert.base64-encode/resource=config

# Chain: decompress then encode
php://filter/read=zlib.inflate|convert.base64-encode/resource=config

# Read without encoding (works for non-PHP files that would not execute)
php://filter/resource=/etc/passwd
```

The [iconv filter](https://www.php.net/manual/en/filters.convert.php) introduced a significant research area in 2022 when it was discovered that chaining multiple iconv character set conversions could generate arbitrary bytes, enabling file writes and RCE through filter chains alone without any file upload. Tools like [php_filter_chain_generator](https://github.com/synacktiv/php_filter_chain_generator) automate this technique, though it falls outside basic LFI and into a more advanced exploitation category.

***

## Files Worth Targeting

Once you have source code disclosure working, these files consistently yield high-value information:

```
config.php          → Database credentials, API keys, secret tokens
.env                → Environment variables with all application secrets
database.php        → Connection strings and credentials
settings.php        → Application configuration, sometimes auth logic
wp-config.php       → WordPress database credentials and secret keys
/etc/passwd         → System users (no filter needed, flat text file)
index.php           → Application entry point, reveals full routing logic
```

Cross-referencing extracted source code with the running application frequently reveals secondary vulnerabilities such as SQL injection, insecure deserialization, or hardcoded admin credentials that are not discoverable through black-box testing alone.
