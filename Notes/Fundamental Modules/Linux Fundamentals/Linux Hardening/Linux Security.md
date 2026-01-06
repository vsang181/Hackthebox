# Linux Security Fundamentals

Every computer system carries some level of risk. Linux systems are generally considered more resilient than many alternatives, largely due to their design and typical deployment models, but they are **not immune to compromise**. Whether you are securing a server or assessing one, you must understand that security is about reducing risk, not eliminating it entirely 

As you work with Linux, your goal should always be to **minimise attack surface**, **limit privilege**, and **detect problems early**.

---

## Keeping the System Up to Date

One of the most effective security controls is also one of the simplest: **keeping the operating system and installed packages up to date**.

Outdated software often contains known vulnerabilities that are trivial to exploit once identified.

You should regularly update the system using:

```text
apt update && apt dist-upgrade
```

This ensures:

* Security patches are applied
* Kernel updates are installed
* Dependency issues are resolved

Be aware that some kernel updates may require a **manual reboot** to take effect. Administrators often overlook this, leaving systems exposed even after updating packages.

---

## Firewalls and Network-Level Controls

If firewall rules are not enforced at the network perimeter, you must rely on **host-based firewalls**.

Linux provides multiple options, including:

* `iptables`
* `nftables`
* Frontends such as `ufw`

These tools allow you to restrict inbound and outbound traffic and significantly reduce exposure. Firewalls should always be configured with a **default-deny mindset**, allowing only what is explicitly required.

---

## Securing SSH Access

SSH is one of the most common entry points into Linux systems, which makes it a frequent target.

At a minimum, SSH should be configured to:

* Disable password-based authentication
* Prevent direct root login
* Use key-based authentication

You should also avoid administering systems as the `root` user whenever possible. Instead, rely on controlled privilege escalation.

---

## Principle of Least Privilege

Access should always be granted based on **what is required**, not convenience.

If a user needs elevated privileges, they should be allowed to run **specific commands** via `sudo`, not granted unrestricted access.

Poorly managed `sudo` permissions are a common cause of privilege escalation. Always prefer narrowly scoped rules over blanket permissions.

---

## Brute-Force Protection with Fail2ban

Tools like **Fail2ban** add an additional defensive layer by monitoring authentication logs and reacting to repeated failures.

Fail2ban can:

* Count failed login attempts
* Temporarily or permanently block offending IP addresses
* Protect services such as SSH, FTP, and web applications

This is particularly effective against automated attacks.

---

## Auditing the System Regularly

Security does not stop once a system is deployed. You should periodically audit the system to identify issues that could lead to compromise.

Key areas to review include:

* Kernel version and patch level
* User and group permissions
* World-writable files and directories
* Misconfigured cron jobs
* Unnecessary or outdated services

Many privilege escalation paths exist purely because something was forgotten or left behind.

---

## Mandatory Access Control: SELinux and AppArmor

Linux supports **mandatory access control (MAC)** frameworks that add another layer of defence beyond standard permissions.

### SELinux

SELinux assigns **labels** to every process, file, and system object. Policies define how these labels are allowed to interact, and the kernel enforces those rules.

This allows for extremely granular control, such as:

* Which processes can read or modify specific files
* Whether a process can execute another program
* Who can append to or move files

SELinux is powerful but complex and requires careful policy management.

---

### AppArmor

AppArmor takes a **profile-based** approach. Instead of labels, applications are confined using predefined profiles that specify allowed behaviour.

It is generally easier to manage than SELinux and is commonly used on Ubuntu systems.

Both systems aim to **limit damage**, even if an application or user is compromised.

---

## Additional Security Tools

Several tools exist to help assess and improve Linux security:

* **Snort** – network intrusion detection
* **chkrootkit** – rootkit detection
* **rkhunter** – system integrity checks
* **Lynis** – security auditing and hardening

These tools do not replace good configuration, but they help identify weaknesses and misconfigurations.

---

## Baseline Hardening Checklist

As a starting point, you should aim to:

* Remove or disable unnecessary services and software
* Eliminate unencrypted authentication mechanisms
* Ensure NTP and syslog are enabled
* Ensure every user has a unique account
* Enforce strong password policies
* Configure password aging and history
* Lock accounts after repeated login failures
* Disable unnecessary SUID and SGID binaries

This list is not exhaustive. Security is **an ongoing process**, not a one-time task.

---

## TCP Wrappers

TCP Wrappers provide a **host-based access control mechanism** for certain network services. They allow you to permit or deny access based on IP address or hostname.

TCP Wrappers rely on two configuration files:

* `/etc/hosts.allow`
* `/etc/hosts.deny`

The system evaluates rules in order, and **the first matching rule is applied**.

---

## `/etc/hosts.allow`

This file defines which hosts or networks are explicitly allowed to access specific services.

Example:

```text
# Allow SSH from the local network
sshd : 10.129.14.0/24

# Allow FTP from a specific host
ftpd : 10.129.14.10

# Allow Telnet from a trusted domain
telnetd : .inlanefreight.local
```

---

## `/etc/hosts.deny`

This file defines which hosts or networks are denied access.

Example:

```text
# Deny all services from a domain
ALL : .inlanefreight.com

# Deny SSH from a specific host
sshd : 10.129.22.22

# Deny FTP from a subnet
ftpd : 10.129.22.0/24
```

---

## Important Limitations of TCP Wrappers

You should keep two key points in mind:

* Rule order matters — the first matching rule wins
* TCP Wrappers **do not replace a firewall**

They only control access to services, not ports, and only work with services compiled to support them.

---

## Final Perspective

Linux security is not about installing a single tool or following a checklist once. It is about **understanding the system**, **reducing exposure**, and **continuously adapting** as threats evolve.

The better you understand how Linux works under the hood, the easier it becomes to secure it—and the easier it becomes to recognise when something is wrong.
