# Medusa

Medusa is a parallel, modular login brute-forcer built for speed across multiple hosts simultaneously. Where Hydra has broader protocol support and is more commonly used for single-target web attacks, Medusa's strength is its threading model and its ability to efficiently handle large-scale multi-host assessments.

***

## Command Structure

```bash
medusa [target_options] [credential_options] -M module [module_options]
```

| Parameter | Purpose | Example |
|---------|---------|---------|
| `-h` / `-H` | Single host or host file | `-h 192.168.1.10` or `-H targets.txt` |
| `-u` / `-U` | Single username or username file | `-u root` or `-U users.txt` |
| `-p` / `-P` | Single password or password file | `-p pass123` or `-P rockyou.txt` |
| `-M` | Module to use | `-M ssh` |
| `-m` | Module-specific options | `-m "DIR:/login"` |
| `-t` | Parallel tasks (threads) | `-t 8` |
| `-f` | Stop after first success on current host | `-f` |
| `-F` | Stop after first success on any host | `-F` |
| `-n` | Non-default port | `-n 2222` |
| `-e ns` | Extra checks: empty password (`n`) and user=pass (`s`) | `-e ns` |
| `-v` | Verbosity level 0-6 | `-v 4` |

***

## Common Attack Scenarios

### SSH

```bash
medusa -h 192.168.0.100 \
       -U usernames.txt \
       -P passwords.txt \
       -M ssh \
       -t 4
```

Keep threads low on SSH for the same reason as Hydra: the cryptographic handshake overhead makes high concurrency counterproductive, and aggressive attempts against SSH will commonly trigger `fail2ban`.

### FTP

```bash
medusa -h 192.168.1.100 \
       -U usernames.txt \
       -P passwords.txt \
       -M ftp \
       -t 10
```

### Multiple Web Servers with Basic Auth

```bash
medusa -H web_servers.txt \
       -U usernames.txt \
       -P passwords.txt \
       -M http \
       -m GET
```

This is where Medusa outshines Hydra for scale. The `-H` flag distributes attempts across every host in the file concurrently, making it well suited for internal network assessments where dozens of devices may share the same default credentials.

### Web Login Forms

```bash
medusa -h www.example.com \
       -U users.txt \
       -P passwords.txt \
       -M web-form \
       -m "FORM:username=^USER^&password=^PASS^:F=Invalid"
```

The `web-form` module syntax is cleaner than Hydra's `http-post-form` three-part string, but it offers slightly less flexibility for complex scenarios involving redirects or token handling.

### Empty and Default Password Checks

```bash
medusa -h 10.0.0.5 \
       -U usernames.txt \
       -e ns \
       -M ssh
```

The `-e ns` flag runs two additional checks before attempting the wordlist:

- `n`: Tests each username with an empty password
- `s`: Tests each username with the username itself as the password (e.g., `admin:admin`, `root:root`)

These two checks cost almost nothing in time and catch a surprising number of real findings on internal networks, particularly on service accounts, database users, and IoT devices.

***

## Medusa Modules Reference

| Module | Protocol | Notable Use |
|--------|---------|------------|
| `ssh` | SSHv2 | Secure remote access |
| `ftp` | FTP | File transfer services |
| `http` | HTTP | Basic auth and GET/POST forms |
| `web-form` | HTTP POST | Login form brute forcing |
| `rdp` | RDP | Windows remote desktop |
| `mysql` | MySQL | Database access |
| `pop3` | POP3 | Email retrieval |
| `imap` | IMAP | Email access |
| `vnc` | VNC | Remote desktop |
| `telnet` | Telnet | Legacy remote access |
| `svn` | SVN | Version control repositories |

***

## Hydra vs Medusa: Practical Differences

| Aspect | Hydra | Medusa |
|--------|-------|--------|
| Protocol support | 50+ | ~20 core modules |
| Multi-host attacks | `-M targets.txt` flag | `-H targets.txt`, native design |
| Web form syntax | Three-part colon string | Cleaner `-m FORM:` syntax |
| CSRF token handling | Not supported natively | Not supported natively |
| Speed on same target | Comparable | Slightly faster on some protocols |
| Community resources | Larger, more documentation | Less but sufficient |
| Empty/default password check | Manual via wordlist | Native `-e ns` flag |

In practice most testers use both. Hydra tends to be the go-to for web application login forms due to its flexibility with the condition string, while Medusa is preferred when running the same attack across a network range of hosts or when the quick `-e ns` default credential check is needed before pulling out a full wordlist.

***

## Verbosity Levels

Medusa's `-v` flag accepts levels from 0 to 6, which is more granular than Hydra's binary `-v`/`-V` options:

| Level | Output |
|-------|--------|
| 0 | Silent, results only |
| 2 | Default, attempts shown |
| 4 | Detailed per-attempt output |
| 6 | Full debug output |

For most assessments `-v 4` gives a good balance of visibility without flooding the terminal. Use level 6 only when debugging why a module is not behaving as expected, as the output volume becomes difficult to parse at that level.
