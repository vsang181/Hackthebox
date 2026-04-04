## Linux Hardening

Hardening is the defender's answer to everything covered in this module. Each privilege escalation technique maps directly to a preventable misconfiguration. A properly hardened system removes low-hanging fruit entirely and forces any attacker toward unreliable kernel exploits, which themselves can be mitigated through timely patching. 
***

## Updates and Patching

Kernel CVEs and outdated service vulnerabilities are the most straightforward escalation paths. Keeping the kernel and all packages current closes them before they can be used. 

```bash
# Debian/Ubuntu: enable automatic unattended security updates
apt install unattended-upgrades
dpkg-reconfigure unattended-upgrades
cat /etc/apt/apt.conf.d/50unattended-upgrades  # review config

# Red Hat/CentOS: equivalent with yum-cron
yum install yum-cron
systemctl enable --now yum-cron
# Config: /etc/yum/yum-cron.conf
# Set: apply_updates = yes

# Manual check for outstanding updates
apt list --upgradable 2>/dev/null
yum check-update
```

***

## Configuration Hardening

### File Permissions and SUID

```bash
# Audit world-writable files (remove any that should not be)
find / -path /proc -prune -o -type f -perm -o+w -print 2>/dev/null

# Audit SUID binaries (remove the bit from anything that does not need it)
find / -perm -4000 -type f 2>/dev/null
chmod u-s /path/to/unnecessary/suid-binary

# Audit SGID binaries
find / -perm -2000 -type f 2>/dev/null

# Audit world-writable directories (check each one is intentional)
find / -type d -perm -o+w 2>/dev/null
```

### Cron Jobs

```bash
# Ensure all cron job scripts and their directories are root-owned and not writable
ls -la /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/
stat /var/spool/cron/

# Verify all commands in crontab use absolute paths
cat /etc/crontab
cat /etc/cron.d/*

# Check cron script permissions
find /etc/cron* -type f -perm -o+w 2>/dev/null  # should return nothing
```

### Library and PATH Security

```bash
# Find world-writable Python module files
find /usr/lib/python* -name "*.py" -perm -o+w 2>/dev/null

# Check for writable RPATH directories used by SUID binaries
find / -perm -4000 -type f 2>/dev/null | while read b; do
    readelf -d "$b" 2>/dev/null | grep -q RPATH && echo "$b"
done

# Ensure env_keep does NOT preserve LD_PRELOAD or LD_LIBRARY_PATH in sudoers
grep -i "LD_PRELOAD\|LD_LIBRARY_PATH" /etc/sudoers /etc/sudoers.d/*
```

### Sensitive Files

```bash
# Credentials should never be world-readable
chmod 640 /etc/shadow
chmod 644 /etc/passwd
chmod 440 /etc/sudoers

# Clean up bash history to prevent credential leakage
cat /dev/null > ~/.bash_history
find /home -name ".bash_history" -exec ls -la {} \;

# Remove cleartext credentials from scripts and config files
grep -r "password\|passwd\|secret\|token" /opt /var/www /home 2>/dev/null | grep -v ".git"
```

***

## User Management

The principle of least privilege applied consistently eliminates a large portion of the attack surface. 

```bash
# List all users with a login shell (reduce this to minimum necessary)
grep -v "nologin\|false" /etc/passwd

# List all users with sudo rights
grep -E "sudo|wheel|admin" /etc/group
sudo -l -U <username>  # check individual user's sudo permissions

# Lock accounts that are not actively needed
usermod -L <username>
passwd -l <username>

# Check for users with UID 0 (should only be root)
awk -F: '($3 == 0) {print}' /etc/passwd

# Enforce password history via PAM (prevent reuse of old passwords)
grep "pam_unix\|remember" /etc/pam.d/common-password
# Add: password required pam_unix.so use_authtok sha512 remember=5

# Set minimum password age to prevent rapid cycling
chage -l <username>
chage -m 7 -M 90 <username>  # min 7 days, max 90 days
```

***

## SELinux / AppArmor

Mandatory access control layers add a second line of defence that operates independently of file permissions. 

```bash
# Check SELinux status (Red Hat-based)
getenforce
sestatus

# Enable enforcing mode
setenforce 1
# Persist in: /etc/selinux/config
# Set: SELINUX=enforcing

# Check AppArmor status (Debian/Ubuntu)
aa-status
systemctl status apparmor

# AppArmor profile for a specific binary
aa-enforce /etc/apparmor.d/usr.sbin.apache2
```

***

## Lynis Audit Tool

Lynis is the most practical tool for getting a baseline security score and a prioritised list of hardening actions. It audits the entire system and maps findings to controls from ISO 27001, PCI-DSS, and HIPAA. 

```bash
# Clone and run (no installation required, no root needed for basic scan)
git clone https://github.com/CISOfy/lynis
cd lynis
./lynis audit system

# Run as root for a complete scan (includes checks skipped in non-privileged mode)
sudo ./lynis audit system

# Quiet run with output to a report file (useful for scheduled audits)
sudo ./lynis audit system --quiet --report-file /var/log/lynis-report.dat

# Read the report for machine-parseable results
grep "warning\[\]" /var/log/lynis-report.dat
grep "suggestion\[\]" /var/log/lynis-report.dat
grep "hardening_index" /var/log/lynis-report.dat
```

Lynis outputs three types of findings: 

- Warnings - critical issues to address immediately (misconfigured cron, bad file permissions)
- Suggestions - lower-priority improvements (GRUB password, core dump limits)
- Hardening Index - an overall score out of 100 showing how much hardening has been applied

***

## Hardening Checklist

This maps each technique from this module to its defensive countermeasure:

| Escalation Vector | Countermeasure |
|---|---|
| Kernel CVEs | Apply kernel and package updates; enable unattended-upgrades   |
| SUID/SGID abuse | Audit and remove unnecessary SUID bits; monitor with Lynis |
| Cron job abuse | Ensure cron scripts are root-owned and not world-writable; use absolute paths |
| Sudo misconfigs | Restrict sudo to specific commands; avoid wildcards; use least privilege |
| LD_PRELOAD / LD_LIBRARY_PATH | Remove `env_keep` for these variables from sudoers |
| Shared object hijacking | Audit RPATH directories; ensure they are not world-writable |
| Python library hijacking | Restrict write access on module directories; avoid `SETENV` in sudoers |
| NFS no_root_squash | Enable `root_squash` on all exports; restrict NFS mounts by IP |
| Docker/LXD group | Restrict group membership; never add users to docker/lxd unless required |
| tmux session hijacking | Restrict permissions on tmux sockets; do not share sockets with groups |
| Passive traffic sniffing | Enforce encrypted protocols; restrict tcpdump capability |
| Password in cleartext files | Scan for hardcoded credentials; use secret management tools |
