# Value Fuzzing

Value fuzzing is the final stage of the parameter enumeration chain. By this point you have a confirmed endpoint, a confirmed parameter name, and a clear error message telling you the value is wrong. The only remaining unknown is the correct value itself.

***

## Building a Custom Wordlist

Pre-made wordlists work well for parameter names because those tend to follow predictable conventions across applications. Values are application-specific, so you often have to generate your own. The Bash one-liner for a sequential numeric range is the fastest approach:

```bash
for i in $(seq 1 1000); do echo $i >> ids.txt; done
```

If you need a wider range, adjust the bounds:

```bash
# 1 to 10,000
for i in $(seq 1 10000); do echo $i >> ids.txt; done

# Python alternative, useful when already in a script
python3 -c "print('\n'.join(str(i) for i in range(1, 1001)))" > ids.txt
```

Both produce an identical file: one number per line, no padding, starting at 1. The choice between Bash and Python comes down to whatever is faster to type in context.

***

## The Value Fuzzing Command

```bash
ffuf -w ids.txt:FUZZ \
     -u http://admin.academy.htb:PORT/admin/admin.php \
     -X POST \
     -d 'id=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fs xxx
```

`FUZZ` is now in the value position rather than the parameter name position. Everything else stays the same as the POST parameter fuzzing command from the previous section. The `-fs xxx` filter excludes the `Invalid id!` response size, so only a response with different content (the actual flag or data) surfaces.

***

## The Full Enumeration Chain

This module has walked through a complete web fuzzing methodology from scratch. Each stage feeds directly into the next:

```
1. Directory fuzzing      ->  Discovers /blog, /forum, /admin
2. Extension fuzzing      ->  Confirms .php
3. Page fuzzing           ->  Finds admin.php, hidden pages
4. Recursive fuzzing      ->  Automates steps 1-3 across all directories
5. Vhost fuzzing          ->  Discovers admin.academy.htb
6. GET parameter fuzzing  ->  Finds deprecated parameter
7. POST parameter fuzzing ->  Finds id parameter
8. Value fuzzing          ->  Finds the valid id value
```

***

## Extracting the Result with Curl

Once ffuf returns a hit, confirm it and retrieve the content:

```bash
curl http://admin.academy.htb:PORT/admin/admin.php \
     -X POST \
     -d 'id=FOUND_VALUE' \
     -H 'Content-Type: application/x-www-form-urlencoded'
```

Replace `FOUND_VALUE` with whichever number ffuf returned. The response should contain the flag or sensitive data rather than the `Invalid id!` message you have been filtering out.

***

## Choosing the Right Wordlist Type for Values

Different parameter types call for different wordlist strategies:

| Parameter Type | Wordlist Approach |
|---------------|------------------|
| Numeric ID | Sequential range with `seq` or Python |
| Username | `seclists/Usernames/` or custom list |
| Token or hash | Format-specific fuzzing (MD5, UUID, etc.) |
| Filename | `seclists/Discovery/Web-Content/` |
| Boolean-style | Small manual list: `true`, `false`, `1`, `0`, `yes`, `no` |
| Enum values | Application-specific, infer from frontend source code |

When you have no clues about the format, start with a small numeric range first since sequential IDs are extremely common in web applications. If that produces no results, inspect the application's JavaScript, HTML source, or any API responses for hints about what format the value should take.
