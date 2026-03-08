# Linux Authentication Process

Linux authentication relies primarily on [Pluggable Authentication Modules (PAM)](https://web.archive.org/web/20220622215926/http://www.linux-pam.org/Linux-PAM-html/Linux-PAM_SAG.html) to manage user authentication, session handling, and password changes across the system. Understanding where and how Linux stores credentials is a prerequisite for credential-based attacks on Linux systems, documented under [MITRE ATT&CK T1003.008](https://attack.mitre.org/techniques/T1003/008/).

## PAM and its Modules

The core PAM module for password handling is `pam_unix.so` (or `pam_unix2.so`), typically located at `/usr/lib/x86_64-linux-gnu/security/` on Debian-based systems. When a user runs `passwd` to change their password, PAM intercepts the request, applies policy checks, and writes the result to the appropriate credential files. PAM also supports additional service modules for [LDAP](https://linux.die.net/man/5/pam_ldap), Kerberos, and mount-based authentication, making it a flexible but complex attack surface. For deeper reading on the full Linux authentication process, the [User Authentication HOWTO](https://tldp.org/HOWTO/pdf/User-Authentication-HOWTO.pdf) from The Linux Documentation Project is a thorough reference.

## /etc/passwd

The `/etc/passwd` file is world-readable and contains one entry per user account, with seven colon-separated fields:

```
htb-student:x:1000:1000:,,,:/home/htb-student:/bin/bash
```

| Field | Value | Description |
|---|---|---|
| Username | `htb-student` | The login name |
| Password | `x` | Password placeholder; actual hash in `/etc/shadow` |
| User ID | `1000` | UID — root is always 0 |
| Group ID | `1000` | Primary group ID |
| [GECOS](https://en.wikipedia.org/wiki/Gecos_field) | `,,,` | Optional comment field (full name, contact info) |
| Home directory | `/home/htb-student` | User's home path |
| Default shell | `/bin/bash` | Shell spawned at login |

The `x` in the password field indicates the hash is stored in `/etc/shadow`. On very old or misconfigured systems, the actual hash may appear here directly — and since `/etc/passwd` is world-readable, any user on the system could read and attempt to crack it offline.

A particularly severe misconfiguration is when `/etc/passwd` is world-writable. In that case, the password field for root can be cleared entirely, allowing passwordless root login:

```bash
# Verify the misconfiguration
ls -la /etc/passwd

# Remove the root password field
head -n 1 /etc/passwd
root::0:0:root:/root:/bin/bash

# Switch to root with no password prompt
su
root@htb[/htb]#
```

## /etc/shadow

The `/etc/shadow` file was introduced specifically to address the risk of world-readable password hashes. It is readable only by root and contains nine colon-separated fields per user:

```
htb-student:$y$j9T$3QSBB6CbHEu...f8Ms:18955:0:99999:7:::
```

| Field | Value | Description |
|---|---|---|
| Username | `htb-student` | Links to the corresponding `/etc/passwd` entry |
| Password | `$y$j9T$3QSBB...` | The hashed password in `$id$salt$hash` format |
| Last change | `18955` | Days since epoch (Jan 1, 1970) of last password change |
| Min age | `0` | Minimum days before password can be changed |
| Max age | `99999` | Maximum days before password must be changed |
| Warning period | `7` | Days before expiry that user is warned |
| Inactivity period | `-` | Days of inactivity before account is disabled |
| Expiration date | `-` | Absolute date the account expires |
| Reserved | `-` | Not currently used |

If the password field contains `!` or `*`, the account is locked for Unix password authentication (other methods such as SSH keys or Kerberos still work). An empty password field means no password is required for login.

### Hash Format

The password field follows the structure `$<id>$<salt>$<hash>`, where the `id` identifies the hashing algorithm used:

| ID | Algorithm |
|---|---|
| `1` | [MD5](https://en.wikipedia.org/wiki/MD5) |
| `2a` | [Blowfish](https://en.wikipedia.org/wiki/Blowfish_(cipher)) |
| `5` | [SHA-256](https://en.wikipedia.org/wiki/SHA-2) |
| `6` | [SHA-512](https://en.wikipedia.org/wiki/SHA-2) |
| `sha1` | [SHA1crypt](https://en.wikipedia.org/wiki/SHA-1) |
| `y` | [Yescrypt](https://github.com/openwall/yescrypt) |
| `gy` | [Gost-yescrypt](https://www.openwall.com/lists/yescrypt/2019/06/30/1) |
| `7` | [Scrypt](https://en.wikipedia.org/wiki/Scrypt) |

Modern Debian-based distributions now default to yescrypt (`$y$`), which is significantly slower to crack than SHA-512 due to its memory-hard design. Older systems running `$6$` (SHA-512) or `$1$` (MD5) hashes are considerably more vulnerable to offline cracking. MD5 in particular should be treated as a near-instant crack with any modern wordlist.

## /etc/security/opasswd

The PAM library can optionally prevent password reuse by maintaining a history of previous password hashes in `/etc/security/opasswd`. This file requires root to read and stores entries in a comma-separated format per user:

```bash
sudo cat /etc/security/opasswd

cry0l1t3:1000:2:$1$HjFAfYTG$qNDkF0zJ3v8ylCOrKB0kt0,$1$kcUjWZJX$E9uMSmiQeRh4pAAgzuvkq1
```

The presence of `$1$` (MD5) hashes here is significant in two ways. First, they are trivial to crack compared to modern algorithms. Second, and more practically, password history reveals patterns — users frequently increment a number, substitute a character, or use a seasonal variation on their previous passwords. Identifying these patterns from opasswd entries can dramatically improve targeted attack wordlists for current credential cracking attempts.

## Cracking Linux Credentials

With root access on a Linux system, both `/etc/passwd` and `/etc/shadow` can be copied and processed offline. The [unshadow](https://github.com/pmittaldev/john-the-ripper/blob/master/src/unshadow.c) utility (included with John the Ripper) merges both files into a single combined format that crackers can parse:

```bash
# Back up the files
sudo cp /etc/passwd /tmp/passwd.bak
sudo cp /etc/shadow /tmp/shadow.bak

# Combine into a crackable format
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

The resulting file can be attacked with Hashcat or John the Ripper. The Hashcat mode depends on the hash algorithm identified by the `$id$` prefix:

```bash
# SHA-512 ($6$) hashes
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes rockyou.txt -o /tmp/cracked.txt

# Yescrypt ($y$) hashes - not supported by hashcat, use JtR instead
john --wordlist=rockyou.txt /tmp/unshadowed.hashes
```

> **Note:** John the Ripper's single crack mode was designed specifically for unshadowed Linux credential files — it uses the GECOS field and username variations as additional seed data for its mangling rules, often cracking simple passwords faster than a straight dictionary attack.

## Hashcat Modes for Linux Shadow Hashes

| Hash Prefix | Algorithm | Hashcat Mode | Relative Speed |
|---|---|---|---|
| `$1$` | MD5 | `500` | Very fast |
| `$5$` | SHA-256 | `7400` | Moderate |
| `$6$` | SHA-512 | `1800` | Slow |
| `$y$` | Yescrypt | Not supported | Use JtR |
| `$7$` | Scrypt | Not supported | Use JtR |
