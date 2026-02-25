# Spawning Interactive Shells

Landing on a system with a limited or non-TTY shell is a routine occurrence during engagements. Commands such as `su`, `sudo`, and any application that reads directly from `/dev/tty` will fail in this environment, blocking privilege escalation attempts before they begin. When Python is unavailable, several alternative methods using native languages and utilities can spawn a fully interactive shell, provided the relevant binary or interpreter exists on the target system.

The shell binary referenced in each method below (`/bin/sh`) can be substituted with any interpreter present on the system, most commonly `/bin/bash` on modern Linux distributions.

## Direct Invocation

Invoke the shell interpreter in interactive mode directly, without requiring any scripting language:

```bash
/bin/sh -i
```

The `-i` flag forces interactive mode. This is the fastest and most dependency-free option available, though it may still produce a limited shell depending on how the session was established.

## Language-Based Methods

Each method below spawns a shell by calling the interpreter from within a scripting language runtime. Confirm availability with `which perl`, `which ruby`, or `which lua` before attempting.

**[Perl](https://www.perl.org/)**, issued as a one-liner from the command line:

```bash
perl -e 'exec "/bin/sh";'
```

Or from within a running Perl script:

```perl
exec "/bin/sh";
```

**[Ruby](https://www.ruby-lang.org/en/)**, from within a running Ruby script:

```ruby
exec "/bin/sh"
```

**[Lua](https://www.lua.org/)**, using the `os.execute` method from within a Lua script:

```lua
os.execute('/bin/sh')
```

## System Utility Methods

These methods leverage utilities almost universally present on Unix/Linux systems, making them reliable fallbacks when no scripting language runtime is available.

**[AWK](https://man7.org/linux/man-pages/man1/awk.1p.html)** is a C-like pattern scanning and processing language present on the vast majority of Unix/Linux systems. Its `BEGIN` block executes before any input is processed, making it suitable for direct command execution:

```bash
awk 'BEGIN {system("/bin/sh")}'
```

**[find](https://man7.org/linux/man-pages/man1/find.1.html)** supports arbitrary command execution via its `-exec` flag. Two approaches are available:

Using `find` to invoke `awk`, which then spawns the shell:

```bash
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```

Using `find`'s `-exec` flag to invoke the shell interpreter directly, then exiting after the first match with `-quit`:

```bash
find . -exec /bin/sh \; -quit
```

If `find` cannot locate the specified file, no shell will be spawned with the first variant. The second variant using `.` (current directory) will match immediately and always succeed.

## VIM

**[VIM](https://www.vim.org/)** can invoke a shell interpreter from within the editor. This is most useful when a restricted environment allows file editing but blocks direct shell invocation:

Directly from the command line:

```bash
vim -c ':!/bin/sh'
```

From within an open VIM session:

```vim
:set shell=/bin/sh
:shell
```

## Execution Permissions Considerations

Once a shell session is established, assess what the current account can actually do before attempting further actions. Check file and binary permissions with:

```bash
ls -la <path/to/fileorbinary>
```

Check what commands the current account can run via `sudo`:

```bash
sudo -l

Matching Defaults entries for apache on ILF-WebSrv:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User apache may run the following commands on ILF-WebSrv:
    (ALL : ALL) NOPASSWD: ALL
```

The `NOPASSWD: ALL` entry in the output above represents a critical misconfiguration: the `apache` account can execute any command as any user without providing a password. This single `sudo -l` result opens a direct path to full root access on the host. The `sudo -l` command requires a stable TTY to return output; running it from an unstable or non-TTY shell will produce no results, which is precisely why upgrading the shell before attempting privilege enumeration is a prerequisite rather than an optional step.
