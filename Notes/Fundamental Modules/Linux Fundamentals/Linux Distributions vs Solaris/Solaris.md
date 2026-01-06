# Solaris Overview and Comparison Notes

Solaris is a **Unix-based operating system** originally developed by Sun Microsystems and later acquired by Oracle. You will mostly encounter it in **enterprise and mission-critical environments**, where stability, performance, and long-term reliability matter more than flexibility or rapid change 

If you ever assess large financial, government, or legacy infrastructure, Solaris is still relevant, even if it is less common than Linux today.

---

## What Solaris Is Designed For

Solaris is built with one clear goal in mind: **enterprise-grade stability at scale**.

It is commonly used for:

* High-availability systems
* Large databases
* Virtualisation platforms
* Data centres and cloud backends
* Banking, finance, and government infrastructure

Solaris includes several features that reflect this focus, such as:

* Advanced service management
* Strong role-based access control
* Tight hardware integration
* Emphasis on fault tolerance and uptime

Unlike most Linux distributions, Solaris is **proprietary**, and its source code is not publicly available.

---

## Solaris vs Linux Distributions

Although Solaris and Linux are both Unix-like, they differ in important ways. You should understand these differences so you do not assume Linux behaviour applies everywhere.

### High-Level Differences

| Area               | Solaris                           | Linux Distributions              |
| ------------------ | --------------------------------- | -------------------------------- |
| Source model       | Proprietary (Oracle)              | Mostly open source               |
| Target audience    | Enterprise / mission-critical     | Broad (desktop to server)        |
| Service management | SMF (Service Management Facility) | systemd / SysV / OpenRC          |
| Access control     | Strong RBAC + MAC                 | DAC + optional MAC               |
| Package management | IPS / SPM                         | APT, YUM, DNF, Pacman            |
| Filesystem focus   | ZFS (native)                      | ext4, XFS, Btrfs, ZFS (optional) |

You should not treat Solaris as “just another Linux distro”. Internals, tooling, and administration models differ in subtle but important ways.

---

## Solaris Directory Structure

Solaris follows a Unix-style directory hierarchy that looks familiar if you know Linux, but there are some Solaris-specific additions.

| Directory     | Purpose                                 |
| ------------- | --------------------------------------- |
| `/`           | Root of the filesystem                  |
| `/bin`        | Essential system binaries               |
| `/boot`       | Bootloader and boot files               |
| `/dev`        | Device files                            |
| `/etc`        | System configuration                    |
| `/home`       | User home directories                   |
| `/kernel`     | Kernel modules and kernel data          |
| `/lib`        | Libraries for system binaries           |
| `/lost+found` | Recovered files after filesystem checks |
| `/mnt`        | Temporary mount points                  |
| `/opt`        | Optional software packages              |
| `/proc`       | Process and kernel information          |
| `/sbin`       | System administration binaries          |
| `/tmp`        | Temporary files                         |
| `/usr`        | User programs and shared data           |
| `/var`        | Logs and variable system data           |

As you can see, much of this mirrors Linux, which helps reduce the learning curve.

---

## System Information Commands

On Linux, you typically use `uname` to retrieve system information.

### Linux Example

```text
uname -a
```

Example output:

```text
Linux ubuntu 5.4.0-1045 #48-Ubuntu SMP Fri Jan 15 10:47:29 UTC 2021 x86_64 GNU/Linux
```

---

### Solaris Equivalent

On Solaris, system information is usually retrieved with `showrev`.

```text
showrev -a
```

Example output:

```text
Hostname: solaris
Kernel architecture: sun4u
OS version: Solaris 10 8/07 s10s_u4wos_12b SPARC
Application architecture: sparc
Hardware provider: Sun_Microsystems
Kernel version: SunOS 5.10 Generic_139555-08
```

Compared to `uname`, `showrev` provides **more enterprise-focused details**, such as patch level, hardware architecture, and vendor information.

---

## Package Management Differences

On Ubuntu and similar Linux systems, package installation is usually done with APT.

### Linux Example

```text
sudo apt-get install apache2
```

---

### Solaris Package Installation

Traditionally, Solaris uses `pkgadd` to install packages.

```text
pkgadd -d SUNWapchr
```

Key differences you should note:

* Older Solaris versions rely heavily on **RBAC**, not `sudo`
* Root access is handled through roles rather than direct privilege escalation
* `sudo` support exists but became common only in Solaris 11

This changes how privilege escalation paths may appear during assessments.

---

## Permission Management

File permission concepts are mostly familiar.

### Changing Permissions

```text
chmod 700 filename
```

This works the same way on Linux and Solaris.

---

### Finding SUID Files

On Linux:

```text
find / -perm 4000
```

On Solaris:

```text
find / -perm -4000
```

The `-` before the permission value is a Solaris-specific nuance. Small differences like this matter during enumeration.

---

## NFS in Solaris

Solaris includes its own NFS implementation, which is configured differently from Linux.

### Sharing a Directory

```text
share -F nfs -o rw /export/home
```

This shares the directory with read/write access.

---

### Mounting an NFS Share

```text
mount -F nfs 10.129.15.122:/nfs_share /mnt/local
```

---

### Persistent NFS Configuration

Solaris stores NFS share definitions in:

```text
/etc/dfs/dfstab
```

Example:

```text
share -F nfs -o rw /export/home
```

When enumerating Solaris systems, always check this file for exposed shares.

---

## Process and File Mapping

On Linux, `lsof` is commonly used to see which files a process has open.

```text
sudo lsof -c apache2
```

---

### Solaris Alternative: `pfiles`

On Solaris, you use `pfiles`.

```text
pfiles `pgrep httpd`
```

This provides similar information, including file descriptors, sockets, and mapped files.

---

## System Call Tracing

Tracing system calls is critical when debugging or analysing application behaviour.

### Linux: `strace`

```text
sudo strace -p `pgrep apache2`
```

---

### Solaris: `truss`

```text
truss ls
```

Example output:

```text
execve("/usr/bin/ls", 0xFFBFFDC4, 0xFFBFFDC8)  argc = 1
...SNIP...
```

Important differences you should remember:

* `truss` can trace **signals**
* `truss` can trace **child processes**
* Output format differs slightly from `strace`

From a security perspective, both tools can expose **sensitive runtime behaviour**, such as file access, network usage, and credential handling.

---

## Why Solaris Still Matters

Even if you rarely use Solaris day-to-day, it appears in:

* Legacy enterprise systems
* Financial infrastructure
* Government environments
* Large-scale storage platforms

Understanding Solaris helps you avoid Linux-centric assumptions and makes you more adaptable during real-world assessments.

Your goal is not to master Solaris, but to **recognise it, enumerate it correctly, and respect its differences**.
