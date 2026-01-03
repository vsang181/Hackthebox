# System Information

Now it is time to get hands-on and start building confidence in the terminal. As you work through this section, remember that help is always available. If you are unsure about a command, you can rely on `-h`, `--help`, or the `man` pages at any time 

Because you will be working with many different Linux systems, it is essential to understand their structure. This includes system details, running processes, network configuration, users and permissions, and directory layout. This knowledge is not only useful for everyday Linux usage, but is also critical during security assessments, where identifying misconfigurations or weak permissions can directly lead to privilege escalation or deeper access.

## Common System Information Commands

Below is a list of commonly used commands that help you gather system information. Most of these tools are available by default on modern Linux systems.

| Command    | Description                               |
| ---------- | ----------------------------------------- |
| `whoami`   | Displays the current username             |
| `id`       | Shows user identity, groups, and IDs      |
| `hostname` | Prints or sets the system hostname        |
| `uname`    | Displays kernel and system information    |
| `pwd`      | Prints the current working directory      |
| `ifconfig` | Displays or configures network interfaces |
| `ip`       | Manages routing, interfaces, and tunnels  |
| `netstat`  | Displays network connections and status   |
| `ss`       | Investigates active sockets               |
| `ps`       | Shows running processes                   |
| `who`      | Displays logged-in users                  |
| `env`      | Prints environment variables              |
| `lsblk`    | Lists block devices                       |
| `lsusb`    | Lists USB devices                         |
| `lsof`     | Lists open files                          |
| `lspci`    | Lists PCI devices                         |

You should become familiar with what each of these commands reveals and when they are useful.

## Logging in via SSH

Secure Shell (SSH) is a protocol that allows you to connect to and execute commands on a remote system securely. On Linux and other Unix-like operating systems, SSH is a standard tool and is widely used by administrators for remote management.

SSH does not provide a graphical interface. Instead, it gives you direct shell access, which makes it fast, efficient, and lightweight. Throughout this path, SSH is used extensively to let you practise commands and techniques in controlled lab environments.

### Connecting to a Remote System

You can connect to a target system using the following syntax:

```text
ssh htb-student@[IP address]
```

Once connected, you will interact with the remote system as if you were sitting in front of it.

## Basic Enumeration Examples

After logging in, the first thing you should do is gather basic situational awareness.

### Hostname

The `hostname` command prints the name of the system you are currently logged into.

```text
hostname
```

This helps confirm which machine you are interacting with, which is especially important when working across multiple hosts.

### Whoami

The `whoami` command returns the username of the current session.

```text
whoami
```

During a penetration test, this is one of the first commands you should run after gaining access. Knowing which user you are operating as allows you to assess privilege level and potential access.

### Id

The `id` command provides more detail than `whoami`. It shows your user ID, group ID, and group memberships.

```text
id
```

From a security perspective, group memberships are extremely important. Membership in groups such as `adm` or `sudo` can indicate access to logs, administrative functions, or the ability to execute commands as root. For defenders, this output is also useful for auditing permissions and identifying unnecessary access.

## Kernel and System Details with `uname`

The `uname` command provides information about the system and kernel. To see available options, you should check the manual page:

```text
man uname
```

One of the most useful flags is `-a`, which prints all available system information:

```text
uname -a
```

This output includes:

* Kernel name
* Hostname
* Kernel release
* Kernel version
* Machine architecture
* Operating system

Understanding this information is critical during security assessments.

### Extracting the Kernel Version

If your goal is to quickly identify potential kernel exploits, you can extract only the kernel release:

```text
uname -r
```

With this value, you can search for known vulnerabilities related to that specific kernel version. This is often one of the first steps when evaluating privilege escalation opportunities on a Linux system.

## Why This Matters

Although it may feel repetitive, studying these commands and understanding their output is extremely valuable. Man pages often reveal functionality you did not expect, and small details in command output can point directly to misconfigurations or weaknesses.

This knowledge is not just about learning Linux. Later in the path, you will use the same information to:

* Identify weak permissions
* Detect insecure configurations
* Discover privilege escalation paths

Take the time to practise these commands, experiment with different flags, and read the documentation. The effort you invest here will pay off repeatedly as you move into more advanced topics.
