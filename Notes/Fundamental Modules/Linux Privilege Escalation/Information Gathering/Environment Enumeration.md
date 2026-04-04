## Linux Environment Enumeration

Thorough environment enumeration after landing a shell gives you the context to choose the right escalation path rather than trying techniques blindly. The goal is to build a complete picture of the system before touching anything that could cause instability or trigger a detection.

***

## Immediate Orientation (First 2 Minutes)

Run these the moment a shell is obtained:

```bash
whoami && id           # Current user and all group memberships
hostname               # System name, often reveals role (web01, db-prod, etc.)
ip a                   # Network interfaces and subnets
sudo -l                # Sudo rights without a password (instant win if NOPASSWD found)
cat /etc/os-release    # Distro and version
uname -a               # Full kernel version and architecture
echo $PATH             # Check for writable directories in the PATH
env                    # Full environment (look for passwords, tokens, API keys)
```

The `id` output tells you immediately if you are in any high-value groups like `sudo`, `docker`, `lxd`, `adm`, or `disk`, each of which has well-documented escalation paths. 
***

## OS and Kernel Details

```bash
cat /etc/os-release          # Distro name, version, codename
uname -a                     # Kernel: version, architecture, build date
cat /proc/version            # Alternative kernel check
lscpu                        # CPU architecture, vendor (useful for exploit compatibility)
cat /etc/shells              # Available shells (note Tmux/Screen if present)
```

Check whether the version is end-of-life. A patched internet-facing Ubuntu LTS is less likely to have a known kernel CVE, but internal systems are frequently years behind on patches. Note the kernel version and cross-reference it with `linux-exploit-suggester` later.

***

## User and Group Enumeration

```bash
cat /etc/passwd                     # All users: UID, GID, home directory, shell
cat /etc/passwd | cut -f1 -d:       # Username list only
grep "sh$" /etc/passwd              # Users with interactive login shells
cat /etc/group                      # All groups and their members
getent group sudo                   # Who has sudo access
getent group docker                 # Docker group (trivial root escalation)
getent group lxd                    # LXD group (container escape to root)
getent group adm                    # adm group can read system logs
ls /home                            # All user home directories
```

### Password Hash Identification

If `/etc/shadow` is readable, or if hashes appear directly in `/etc/passwd` (common on embedded devices and routers), identify the algorithm from the prefix before cracking: 

| Prefix | Algorithm |
|---|---|
| `$1$` | MD5 |
| `$2a$` | Blowfish/BCrypt |
| `$5$` | SHA-256 |
| `$6$` | SHA-512 |
| `$7$` | Scrypt |
| `$argon2i$` | Argon2 |

SHA-512 (`$6$`) is the most common on modern Linux systems. Submit crackable hashes to `hashcat` with `-m 1800` for SHA-512 crypt. 

***

## Network Context

```bash
ip a                         # All interfaces and IPs (look for multiple NICs suggesting pivot targets)
route                        # Routing table (other reachable subnets)
netstat -rn                  # Same as above
cat /etc/resolv.conf         # DNS servers (internal DNS = Active Directory is likely present)
arp -a                       # ARP cache (hosts the target has recently communicated with)
ss -tulpn                    # Listening services and associated PIDs
```

The ARP cache is particularly valuable. Hosts in the cache represent active connections or recent communication, and any recovered SSH keys should be tried against these addresses. An internal DNS server in `/etc/resolv.conf` is a strong signal of a domain environment where NTLM hashes can be captured or Kerberoasting is possible. 

***

## Home Directory and Credential Hunting

```bash
ls /home                                        # All user home directories
ls -la /home/<user>/                            # Check individual home dirs
cat /home/*/.bash_history 2>/dev/null           # Command history for all users
find /home -name "*.json" -readable 2>/dev/null # Config files
find /home -name ".ssh" -type d 2>/dev/null     # SSH key directories
find / -name "id_rsa" 2>/dev/null               # Private SSH keys anywhere
find / -name "authorized_keys" 2>/dev/null      # Authorised keys files
```

Command history is one of the most overlooked sources of credentials. Developers and admins frequently pass passwords as command-line arguments to `mysql`, `psql`, `curl`, `ssh`, or `smbclient`, and these persist in `.bash_history` until deliberately cleared. 

***

## Filesystem and Storage

```bash
df -h                                   # Mounted filesystems and usage
lsblk                                   # All block devices including unmounted partitions
cat /etc/fstab | grep -v "#" | column -t  # Configured mounts (look for credential options)
grep -i "password\|username\|credential" /etc/fstab  # Credentials in mount options
mount | grep -v "tmpfs\|cgroup\|proc"    # Currently mounted non-system filesystems
lpstat                                  # Printer queues (sometimes contain sensitive documents)
```

An unmounted partition in `lsblk` that is not in `df -h` output is worth investigating. It may contain backups, database files, or application data not accessible from the running system. 

***

## Hidden Files and Temporary Directories

Hidden files (prefixed with `.`) often contain credentials, notes, or leftover artifacts from previous administrator activity. 

```bash
# Hidden files owned by current user or in accessible paths
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep <username>

# Hidden directories
find / -type d -name ".*" -ls 2>/dev/null

# Check temp directories for leftover scripts and artifacts
ls -la /tmp /var/tmp /dev/shm
```

Temporary directory behaviour differences: 

| Directory | Data Retention | Cleared on Reboot |
|---|---|---|
| `/tmp` | Up to 10 days | Yes |
| `/var/tmp` | Up to 30 days | No |
| `/dev/shm` | Until unmounted | Yes |

`/var/tmp` is worth checking carefully because scripts, partial downloads, and credential files can persist there across reboots without the owner noticing.

***

## Configuration File and Secret Searching

```bash
find / -name "*.conf" -readable 2>/dev/null
find / -name "*.config" -readable 2>/dev/null
find / -name ".env" -readable 2>/dev/null
find / -name "wp-config.php" 2>/dev/null
find / -name "*.ini" -readable 2>/dev/null
grep -r "password" /etc/ 2>/dev/null | grep -v "#"
grep -rE "password|passwd|secret|token|api_key" /var/www/ 2>/dev/null
```

Any password discovered at this stage should be immediately tested against every user account on the system using `su`, as well as against SSH on the current and any ARP-cache hosts. Password reuse is consistent and reliable across real environments.

***

## Active Defences Check

Knowing what is monitoring the system affects how you operate. High-noise activities like running `linpeas.sh` at full speed may trip Fail2ban or trigger Snort rules. 

```bash
cat /etc/apparmor.d/ 2>/dev/null     # AppArmor profiles
sestatus 2>/dev/null                 # SELinux status and mode
iptables -L 2>/dev/null              # Firewall rules (requires root, but attempt anyway)
ufw status 2>/dev/null               # UFW status
systemctl status fail2ban 2>/dev/null  # Fail2ban active?
find /var/log -name "*.log" -readable 2>/dev/null  # Accessible log files
```

If AppArmor or SELinux is in enforcing mode, certain exploitation techniques may fail or produce errors that look like permission denials rather than exploitation failures.
