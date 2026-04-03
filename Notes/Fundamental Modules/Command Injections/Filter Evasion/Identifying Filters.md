# Identifying Filters and WAFs

Before attempting any bypass, the first task is characterising exactly what is being blocked. A filter that blocks specific characters requires different evasion than one that blocks command keywords, and a WAF has different behaviour from application-level PHP code. Identifying the filter type determines which bypass technique to reach for first.

***

## Distinguishing Filter Types by Error Response

The error message and its location in the response tells you where the blocking is happening: 

**Application-level filter (PHP blacklist):** Error appears in the normal output area of the application, in the same location where successful results would appear. The HTTP response code is usually 200 and the page structure is unchanged.

**WAF block:** Typically produces a different page entirely, often with a reference number, your IP address, and a generic "access denied" message. Some WAFs return 403 or 406 status codes rather than 200.

**Front-end validation:** Error appears without any network request being made, confirmed by checking the browser Network tab and seeing no outbound HTTP traffic.

In code, a typical PHP blacklist filter looks like this:

```php
$blacklist = ['&', '|', ';', ' ', '/', 'sh', 'bash', 'whoami', ...];
foreach ($blacklist as $character) {
    if (strpos($_POST['ip'], $character) !== false) {
        echo "Invalid input";
        exit;
    }
}
```

This checks for the presence of each blacklisted string anywhere in the input. The flaw is that it checks for exact string matches, which means encoding, case changes, or character substitutions that the shell still interprets correctly will bypass it. 

***

## Systematic Character Identification

Once you know blocking is happening server-side, isolate what is triggering it by binary reduction: start with your full payload and remove components one at a time until the request succeeds, then add them back one at a time to find the exact trigger. 

```
Full payload: 127.0.0.1; whoami
Result: blocked

Test 1: 127.0.0.1;
Result: blocked → semicolon is blacklisted

Test 2: 127.0.0.1 whoami
Result: blocked → space or "whoami" is blacklisted

Test 3: 127.0.0.1
Result: allowed → baseline passes

Test 4: 127.0.0.1%0awhoami  (newline instead of semicolon)
Result: check if newline also blacklisted

Test 5: 127.0.0.1 test
Result: check if space alone is blacklisted (no command keyword)
```

This systematic reduction tells you precisely which characters are in the blacklist, which avoids wasting bypass attempts on the wrong filter. 
***

## Fuzzing Characters Systematically with Burp

Rather than testing manually, Burp Intruder can fuzz all injection characters in a single run. Set the position on the injection character and load a character list:

```
Positions:  127.0.0.1§;§whoami
Payloads:   ; & | && || %0a %0d \n ` $()
```

Sort results by response length or content. Responses matching the baseline "allowed" length indicate characters that passed the filter, while responses matching the "Invalid input" length indicate blocked characters.

***

## Identifying Blocked Commands

After identifying which operators pass the character filter, the next question is whether specific command keywords are also blacklisted. A command filter typically looks like:

```php
$blacklist = ['whoami', 'cat', 'ls', 'id', 'pwd', 'wget', 'curl', '/etc/passwd'];
foreach ($blacklist as $word) {
    if (strpos($_POST['ip'], $word) !== false) {
        echo "Invalid input";
    }
}
```

Test this by substituting the command with a clearly non-dangerous command that would produce output if it ran: 
```bash
# If semicolon passes but whoami is blocked:
127.0.0.1; hostname    # test if hostname is blocked
127.0.0.1; echo test   # test if echo is blocked
127.0.0.1; id          # test if id is blocked separately
127.0.0.1; ls          # test if ls is blocked
```

Each test isolates one variable. If `hostname` passes but `whoami` is blocked, you know the filter is a keyword blacklist rather than a blanket "no commands" rule.

***

## Reading the Filter's Scope

Different error messages can hint at what exactly triggered the block, though developers often use generic messages deliberately: 

| Error Message | Likely Cause |
|--------------|--------------|
| "Invalid input" | Character or keyword blacklist match |
| "Only IPs allowed" | Format validation (regex check) failed |
| "Access denied" with new page | WAF intercept |
| Request times out with no response | WAF dropping connection entirely |
| 403 Forbidden | WAF or server-level access control |
| 200 with empty response | Filter stripping blacklisted content rather than blocking |

The "stripping" case (last row) is particularly important. Some filters remove blacklisted characters rather than rejecting the request, which means the command reaches the back-end with the offending characters removed. This creates a different bypass path: construct a payload where removing the filtered character still leaves valid commands. For example, if the filter strips `&`, the payload `127.0.0.1;;&;;&;;whoami` becomes `127.0.0.1;;; ;;whoami` after stripping, which still chains commands via the remaining semicolons. 

The identification phase is not about finding one bypass immediately. It is about building a complete picture of what the filter blocks, how it blocks it (reject vs strip), and whether it operates at the character level, keyword level, or both, so that the most appropriate bypass technique can be selected methodically rather than guessed.
