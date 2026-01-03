# Network Services

Working with network services is a core part of understanding Linux systems. Many services exist specifically to **enable communication between hosts**, whether that is for remote administration, file sharing, web access, or secure connectivity. From a security perspective, every running network service represents **potential attack surface**, which makes it essential that you understand how these services work, how they are configured, and how they are commonly misused 

As you move through this section, you should always ask yourself:

* What service is running?
* What protocol does it use?
* Is the communication encrypted?
* Who is allowed to access it?
* What could go wrong if it is misconfigured?

A lack of awareness around network services often leads to serious security failures.

---

## Why Network Services Matter

Network services are designed to perform specific tasks, many of which involve **remote access**. If these services are poorly configured or misunderstood, sensitive data can be exposed.

For example, if a user connects to a remote system using **unencrypted FTP**, credentials can be captured directly from network traffic. This is not a theoretical risk; it happens frequently in real environments. From an administrative perspective, this represents a serious oversight. From a penetration testing perspective, it represents a clear opportunity.

You will not learn every possible network service here. Instead, you will focus on the most important ones that appear regularly in real-world environments.

---

## Secure Shell (SSH)

**Secure Shell (SSH)** is one of the most widely used network protocols on Linux systems. It provides encrypted communication for remote login, command execution, and file transfer.

To connect to a Linux host using SSH, an SSH server must be installed and running on the target system.

### OpenSSH

The most common SSH implementation is **OpenSSH**, which is free, open source, and installed by default on many Linux distributions.

Administrators rely on OpenSSH to:

* Manage systems remotely
* Execute commands securely
* Transfer files safely
* Avoid credential exposure on the network

---

### Installing OpenSSH

```text
sudo apt install openssh-server -y
```

---

### Checking SSH Server Status

```text
systemctl status ssh
```

Example output:

```text
● ssh.service - OpenBSD Secure Shell server
   Active: active (running)
```

This confirms that the SSH daemon (`sshd`) is running and ready to accept connections.

---

### Connecting to a Remote System via SSH

```text
ssh cry0l1t3@10.129.17.122
```

On first connection, you will be asked to trust the host key. Once accepted, the key is stored in your `known_hosts` file to prevent future man-in-the-middle attacks.

SSH can also be used for **port forwarding and tunnelling**, allowing you to route other network traffic through an encrypted channel. This is particularly useful during assessments where direct access to internal services is restricted.

---

### SSH Configuration

OpenSSH is configured via:

```text
/etc/ssh/sshd_config
```

This file controls settings such as:

* Authentication methods (passwords vs keys)
* Maximum concurrent connections
* Root login permissions
* Port forwarding options

Misconfiguration here is a common source of security issues, so changes should always be made carefully.

---

## Network File System (NFS)

**Network File System (NFS)** allows files stored on a remote system to be accessed as if they were local. It is commonly used for centralised storage and shared resources across Linux (and sometimes Windows) environments.

Administrators use NFS to:

* Share project directories
* Centralise data storage
* Simplify collaboration

From a security perspective, NFS is particularly interesting because **misconfigured permissions can lead to privilege escalation**.

---

### Installing NFS

```text
sudo apt install nfs-kernel-server -y
```

---

### Checking NFS Server Status

```text
systemctl status nfs-kernel-server
```

---

### NFS Configuration

NFS exports are defined in:

```text
/etc/exports
```

This file specifies:

* Which directories are shared
* Which hosts can access them
* What permissions are applied

Common NFS options include:

| Option           | Description                       |
| ---------------- | --------------------------------- |
| `rw`             | Read and write access             |
| `ro`             | Read-only access                  |
| `no_root_squash` | Remote root keeps root privileges |
| `root_squash`    | Remote root mapped to normal user |
| `sync`           | Writes committed before returning |
| `async`          | Faster but less safe writes       |

The `no_root_squash` option is especially dangerous if exposed to untrusted clients.

---

### Creating and Sharing an NFS Directory

```text
mkdir nfs_sharing
echo '/home/cry0l1t3/nfs_sharing hostname(rw,sync,no_root_squash)' >> /etc/exports
```

---

### Mounting an NFS Share

```text
mkdir ~/target_nfs
mount 10.129.12.17:/home/john/dev_scripts ~/target_nfs
```

Once mounted, the files behave exactly like local files. In certain scenarios, NFS misconfigurations can be exploited to gain elevated privileges on the remote system.

---

## Web Servers

Web servers are a **primary target** during penetration tests. They deliver content using HTTP or HTTPS and often host applications, APIs, or internal tools.

Common Linux web servers include:

* Apache
* Nginx
* Lighttpd
* Caddy

Apache remains one of the most widely deployed options.

---

### Installing Apache

```text
sudo apt install apache2 -y
```

---

### Apache Configuration

Apache’s global configuration is located at:

```text
/etc/apache2/apache2.conf
```

Example directory configuration:

```text
<Directory /var/www/html>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

This configuration allows full access to `/var/www/html`. Files placed here can be downloaded remotely using tools such as `curl` or `wget`, making Apache useful for file transfer during assessments.

---

### Python Web Server

For quick file hosting, Python provides a lightweight HTTP server.

Start a server in the current directory:

```text
python3 -m http.server
```

Host a specific directory:

```text
python3 -m http.server --directory /home/cry0l1t3/target_files
```

Change the port:

```text
python3 -m http.server 443
```

This is extremely useful when you need a fast, disposable file server without installing additional software.

---

## Virtual Private Networks (VPN)

A **VPN** creates an encrypted tunnel between your system and another network, making it appear as if you are physically inside that network.

VPNs are used to:

* Secure remote access
* Protect data in transit
* Hide internal infrastructure from the public internet

---

### OpenVPN

**OpenVPN** is one of the most popular VPN solutions on Linux. It is open source, flexible, and widely supported.

Install OpenVPN:

```text
sudo apt install openvpn -y
```

Configuration is handled via:

```text
/etc/openvpn/server.conf
```

---

### Connecting to a VPN

Using a provided configuration file:

```text
sudo openvpn --config internal.ovpn
```

Once connected, you can interact with internal systems as if you were on the same network. This is essential for penetration testers when direct access is not possible.

---

## Why This Matters

Network services are where systems interact with the outside world. Misunderstanding them leads to:

* Credential exposure
* Unauthorised access
* Data leakage
* Privilege escalation

As you continue learning, always enumerate running services, inspect their configurations, and question whether they are **necessary, secure, and correctly configured**. Mastering network services gives you both defensive insight and offensive leverage.
