## Linux Services and Internals Enumeration

This phase goes deeper than basic user and file checks. The goal is to understand what is actually running on the system, what software is installed, how processes communicate, and what connections exist to other systems. This data feeds directly into identifying attack surfaces that initial enumeration misses.

***

## Network Internals

### Interfaces and Routing

```bash
ip a                          # All interfaces, IPs, subnets
ifconfig                      # Alternative if net-tools is installed
ip route                      # Routing table (look for additional subnets)
route                         # Alternative routing table view
netstat -rn                   # Numeric routing table
cat /etc/hosts                # Static hostname entries (can reveal internal infrastructure)
cat /etc/resolv.conf          # DNS servers (internal DNS = Active Directory likely present)
```

Multiple network interfaces on the same host (`ens192`, `ens224`, `eth1`, etc.) signal a dual-homed machine positioned at a network boundary.  This makes it a pivot point to reach subnets not accessible from your attack host. Document every interface and its subnet immediately. 

### Active Connections and Open Ports

```bash
ss -tulpn                     # All listening TCP/UDP sockets with process names
netstat -tulpn                # Same on older systems
ss -anp                       # All connections including established
```

Pay attention to services listening only on `127.0.0.1`. These are internal services not reachable from outside that may be running as root or a privileged service account, and they become reachable once you have a shell on the box.

***

## User Activity and Session Information

### Who Is Currently on the System

```bash
w                             # Currently logged-in users, what they are running
who                           # Simpler view of logged-in users
finger <user>                 # Detailed user info if finger is installed
```

Active users matter for two reasons. First, their running processes may hold credentials in memory. Second, if another user is running `sudo su`, you know credential reuse attempts against their account are worth prioritising.

### Login History

```bash
lastlog                       # Last login time for every account on the system
last                          # Login history from /var/log/wtmp
last -a                       # Same with resolved hostnames
```

`lastlog` identifies which accounts actually see interactive use versus service accounts that have never logged in. Frequently accessed accounts suggest active admins whose credential material may be lurking in history files or configuration. 

### Command History

```bash
history                        # Current user's command history
cat ~/.bash_history
cat /home/*/.bash_history 2>/dev/null

# Find all history files on the system
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
```

Admins regularly type passwords directly on the command line as arguments to `mysql -u root -p`, `smbclient //server/share -U admin%password`, `curl -u user:pass`, and similar. These persist verbatim in history files. 

***

## Cron Jobs and Scheduled Tasks

```bash
crontab -l                        # Current user's cron jobs
crontab -l -u <user>              # Another user's cron jobs (if permitted)
cat /etc/crontab                  # System-wide cron schedule
ls -la /etc/cron.daily/
ls -la /etc/cron.hourly/
ls -la /etc/cron.weekly/
ls -la /etc/cron.d/
cat /var/spool/cron/crontabs/* 2>/dev/null

# Watch for cron jobs running in real time (use pspy)
./pspy64
```

Static cron inspection misses jobs that appear and disappear rapidly. `pspy` monitors process creation without requiring root and is the best way to catch cron jobs that run frequently.  When you find a script executed by a root cron job, immediately check: 

1. Is the script world-writable?
2. Does it use relative paths that can be hijacked via `$PATH` manipulation?
3. Does it call other scripts or binaries you can control?

***

## Installed Packages and Version Analysis

```bash
# Debian/Ubuntu
apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee installed_pkgs.list

# RPM-based (RHEL, CentOS, Fedora)
rpm -qa | tee installed_pkgs.list

# Check sudo version specifically
sudo -V
```

Sudo itself has had multiple privilege escalation CVEs. Notable ones include CVE-2021-3156 (Baron Samedit, heap overflow, affects sudo < 1.9.5p2) and CVE-2019-14287 (sudo < 1.8.28 with `(ALL, !root)` rule bypass). Cross-reference the version from `sudo -V` against known CVEs before moving on.

### Automated GTFOBins Cross-Reference

After building `installed_pkgs.list`, this one-liner compares it against the GTFOBins API and flags any installed binaries that have documented abuse paths:

```bash
for i in $(curl -s https://gtfobins.github.io/api.json | jq -r '.executables | keys[]'); do
    if grep -q "$i" installed_pkgs.list; then
        echo "Check GTFO for: $i"
    fi
done
```

This surfaces binaries you might overlook. A system may have `awk`, `python3`, `perl`, `dd`, `tar`, or `cp` installed as standard packages, all of which have GTFOBins entries for SUID abuse, sudo abuse, or file read/write operations. 

***

## Process Analysis

```bash
ps aux                              # All processes with CPU/memory
ps aux | grep root                  # Processes running as root specifically
ps -ef                              # Full format process list
ps -ef --forest                     # Process tree (parent/child relationships)
```

A third-party service or custom binary running as root is the most valuable find in this output. Anything that is not a standard OS component and is running as root should be noted and researched for known CVEs or misconfigurations. 

### Reading Process Arguments from /proc

```bash
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"
```

The `/proc` filesystem is a virtual kernel interface that contains runtime information about every running process.  Reading `cmdline` for each PID dumps the full command line used to start the process, including any credentials passed as arguments at launch time. This catches passwords that have since been cleared from history. 

***

## System Call Tracing with strace

`strace` attaches to a process and prints every system call it makes, including file opens, network connections, and data reads and writes.  This is useful in two scenarios: 

1. Tracing a binary you cannot read to understand what files or credentials it accesses
2. Attaching to a running process to observe live credential passing

```bash
# Trace a binary from launch
strace /usr/local/bin/someapp 2>&1 | grep -iE "open|read|password|pass"

# Attach to an already-running process (requires same UID or root)
strace -p <PID> 2>&1 | grep -iE "read|write|password"

# Trace SUID binary to find missing shared object (for shared object injection)
strace <suid_binary> 2>&1 | grep -iE "open|access|no such file"
```

The last variant is particularly useful for shared object injection: if an SUID binary tries to open a `.so` file from a world-writable path and fails, you can place a malicious shared object there and have it loaded as root.

***

## Scripts, Configs, and Binaries

```bash
# Shell scripts on the system
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

# All readable config files
find / -type f \( -name "*.conf" -o -name "*.config" \) -exec ls -l {} \; 2>/dev/null

# Compiled binaries in non-standard locations
ls -l /usr/local/bin /usr/local/sbin /opt/

# Available attack and scripting tools
which gcc cc python3 python perl ruby netcat nc ncat wget curl tcpdump nmap socat 2>/dev/null
```

Tools like `gcc`, `python3`, `perl`, and `gcc` present on a target mean you can compile or run exploit code locally without needing to transfer a binary. `netcat`, `socat`, and `curl` are useful for data exfiltration, reverse shells, and pivoting. 

***

## Putting It Together: Enumeration Flow

| Stage | Key Commands | What You Are Looking For |
|---|---|---|
| Network | `ip a`, `ss -tulpn`, `arp -a` | Additional subnets, internal services, pivot targets |
| Sessions | `w`, `lastlog`, `history` | Active admins, credential patterns, known hosts |
| Cron | `crontab -l`, `ls /etc/cron.*`, `pspy` | Root-executed writable scripts |
| Packages | `apt list`, `sudo -V`, GTFOBins loop | Vulnerable versions, abusable binaries |
| Processes | `ps aux`, `/proc/*/cmdline`, `strace` | Root services, credentials in memory/arguments |
| Scripts | `find` for `.sh`, `.conf`, `.env` | Hardcoded credentials, weak permissions |

Every piece of information gathered here feeds a specific attack in later stages. The kernel version informs exploit selection. The cron job list informs path hijacking. The installed package list informs binary abuse. None of this data is useful in isolation but together it builds the complete picture needed to choose the right escalation path with the least risk.
