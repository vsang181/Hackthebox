## Linux Remote Management Protocols

Linux servers are frequently administered remotely. This is essential for day-to-day operations, especially when teams are distributed or when systems are hosted in data centres and cloud environments. Remote management tooling is also a common target during penetration tests: these services are often Internet-facing, widely deployed internally, and can provide direct access when misconfigured or protected by weak credentials.

This section covers three remote management categories commonly encountered during assessments:

* **SSH** (secure remote shell and tunnelling)
* **Rsync** (file synchronisation and mirroring)
* **R-Services** (legacy remote access commands)

---

## SSH

Secure Shell (SSH) provides an encrypted channel between two systems over an untrusted network, typically on **TCP/22**. SSH supports interactive shell access, remote command execution, file transfers (SCP/SFTP), and port forwarding. Because SSH is widely available across Unix-like systems and commonly used for remote administration, it is one of the most important services to understand and assess.

Two protocol versions exist: **SSH-1** and **SSH-2**. SSH-2 is the modern standard and significantly more secure. SSH-1 is considered unsafe (including susceptibility to certain man-in-the-middle scenarios) and should be disabled wherever possible.

### Authentication Methods

OpenSSH supports multiple authentication mechanisms, including:

* Password authentication
* Public key authentication
* Host-based authentication
* Keyboard-interactive authentication
* Challenge-response authentication
* GSSAPI authentication

In real environments, **password authentication** and **public key authentication** are the most common.

### Public Key Authentication

Public key authentication avoids repeatedly sending passwords to a remote server. Instead, it relies on an asymmetric key pair:

* The **private key** remains on the client machine and should be protected with a strong passphrase.
* The **public key** is placed on the server (typically in `~/.ssh/authorized_keys`).

When a client connects, the server verifies that the client can prove possession of the private key corresponding to the stored public key. This provides strong authentication without exposing the account password during login attempts.

### Default Configuration

OpenSSH server behaviour is controlled via `sshd_config`. Many settings are commented out by default and must be explicitly enabled or adjusted.

Example excerpt:

```
Include /etc/ssh/sshd_config.d/*.conf
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding yes
PrintMotd no
AcceptEnv LANG LC_*
Subsystem       sftp    /usr/lib/openssh/sftp-server
```

### Dangerous Settings

Even though SSH is secure by design, poor configuration can create straightforward attack paths.

Common risky settings include:

* **PasswordAuthentication yes**: enables password login, increasing exposure to brute force and password spraying.
* **PermitEmptyPasswords yes**: allows empty passwords (high severity misconfiguration).
* **PermitRootLogin yes**: permits direct root login (removes an important barrier).
* **Protocol 1**: enables legacy SSH-1 (unsafe).
* **X11Forwarding yes**: expands attack surface; historically associated with vulnerabilities in older versions.
* **AllowTcpForwarding yes** / **PermitTunnel**: may allow pivoting or bypass of network segmentation if not controlled.
* **DebianBanner yes**: can leak platform details and help attackers tailor exploitation attempts.

### Footprinting SSH

A quick way to fingerprint an SSH server is to capture the banner and supported algorithms. `ssh-audit` is useful for identifying weak ciphers, host key issues, and general hardening gaps.

Example usage:

```
./ssh-audit.py <target>
```

The verbose SSH client output can also reveal which authentication methods are enabled:

```
ssh -v user@<target>
```

If you want to focus on password authentication (for example, when testing credential reuse), you can request a specific method:

```
ssh -v user@<target> -o PreferredAuthentications=password
```

---

## Rsync

Rsync is a fast and efficient utility for file synchronisation and copying, commonly used for backups, mirroring, and deployment workflows. It is best known for its **delta-transfer** behaviour, which transfers only differences when a destination already has an older copy.

Rsync commonly runs as a daemon on **TCP/873**, but it can also operate over SSH (often preferred for secure transport).

### Footprinting Rsync

To detect rsync daemon exposure:

```
sudo nmap -sV -p 873 <target>
```

If the port is open, you can often request a module listing using a simple TCP connection:

```
nc -nv <target> 873
#list
```

If the server exposes modules, you can enumerate a specific module:

```
rsync -av --list-only rsync://<target>/<module>
```

### Why Rsync Matters in Assessments

Misconfigured rsync shares may allow:

* Unauthenticated listing of directories
* Unauthenticated download of files
* Exposure of sensitive material such as configuration files, credentials, deployment artifacts, and SSH keys

If you already have credentials from elsewhere in the environment, rsync is also worth checking for reuse or weak access controls.

---

## R-Services (rlogin, rsh, rexec, rcp)

R-Services are legacy Unix remote access utilities originally developed at UC Berkeley. They predate SSH and are fundamentally unsafe by modern standards because they transmit data **unencrypted** and rely heavily on **trust relationships** rather than strong authentication.

Commonly associated ports:

* **TCP/512** (rexec)
* **TCP/513** (rlogin)
* **TCP/514** (rsh/rcp)

These services are most commonly encountered on older commercial Unix environments such as Solaris, HP-UX, and AIX, but they still occasionally appear in internal networks.

### Common r-commands

| Command | Service daemon | Port | Transport | Description                                                       |
| ------- | -------------- | ---- | --------- | ----------------------------------------------------------------- |
| rcp     | rshd           | 514  | TCP       | Remote file copy (similar to `cp`, but over the network)          |
| rsh     | rshd           | 514  | TCP       | Remote command execution/shell based on trust files               |
| rexec   | rexecd         | 512  | TCP       | Remote command execution; can use username/password (unencrypted) |
| rlogin  | rlogind        | 513  | TCP       | Remote login to Unix-like systems (similar to telnet)             |

### Trusted Relationships: `/etc/hosts.equiv` and `.rhosts`

The key risk in r-services is how they handle access control. Trust can be granted by:

* `/etc/hosts.equiv` (system-wide trust rules)
* `~/.rhosts` (per-user trust rules)

Entries can allow access based on `<host> <user>` pairs, and the `+` wildcard can dangerously permit broad access. Misconfigurations here can allow login **without a password**.

### Footprinting R-Services

Scan for open ports:

```
sudo nmap -sV -p 512,513,514 <target>
```

If misconfigured trust exists, `rlogin` may succeed without credentials:

```
rlogin <target> -l <user>
```

### Useful Enumeration: rwho and rusers

If enabled, rwho/rusers can reveal logged-in users and active sessions, which can help with username discovery and targeting.

Examples:

* `rwho` can list users on the network.
* `rusers -al <host>` provides detailed session information.
