# Network Configuration

As a penetration tester, you will constantly interact with networks. Being comfortable with **configuring and managing network settings on Linux** gives you control over your testing environment and allows you to adapt quickly when things do not work as expected. This knowledge helps you build labs, observe traffic, troubleshoot connectivity, and identify weaknesses that can later be abused during an assessment 

You should think of network configuration as learning how to control **how your machine communicates**, not just whether it can connect or not.

---

## What Network Configuration Covers

In practice, network configuration includes:

* Managing network interfaces
* Assigning IP addresses and routes
* Configuring DNS resolution
* Understanding core protocols such as TCP/IP, DNS, DHCP, and FTP
* Diagnosing and fixing connectivity problems

You will also need to understand the difference between **wired, wireless, and virtual interfaces**, as well as how traffic flows between networks.

---

## Network Access Control (NAC)

Network access control defines **who is allowed to access what** on a network. From a penetration testing perspective, NAC is important because it is often misconfigured or partially enforced.

You will mainly encounter three access control models:

| Model    | Description                                           |
| -------- | ----------------------------------------------------- |
| **DAC**  | Resource owners decide who can access their resources |
| **MAC**  | The operating system enforces access rules            |
| **RBAC** | Permissions are assigned based on roles               |

Linux systems often combine these models with additional security frameworks.

---

## NAC Mechanisms on Linux

On Linux, network access control is commonly implemented using:

* **SELinux** – kernel-level mandatory access control
* **AppArmor** – profile-based access control for applications
* **TCP Wrappers** – IP-based access restrictions for services

You will explore these in more detail later, but for now, remember that NAC exists to **limit impact**, even if a system is partially compromised.

---

## Monitoring Network Activity

Monitoring traffic is essential for both offence and defence. By observing network behaviour, you can identify:

* Cleartext credentials
* Unexpected services
* Data leaks
* Suspicious connections

Common tools you will see include:

* `syslog` and `rsyslog`
* `ss`
* `lsof`
* The ELK stack (Elasticsearch, Logstash, Kibana)

Traffic analysis often reveals information that is not obvious from configuration files alone.

---

## Configuring Network Interfaces

On modern Linux systems, network interfaces are typically managed using the `ip` command. The older `ifconfig` command is deprecated but still widely encountered.

### Viewing Network Interfaces

Using `ifconfig`:

```text
ifconfig
```

Using `ip` (recommended):

```text
ip addr
```

From this output, you should be able to identify:

* Interface names (for example, `eth0`, `eth1`, `lo`)
* IPv4 and IPv6 addresses
* Netmasks and broadcast addresses
* Interface state (UP or DOWN)

This is your starting point whenever network connectivity is not behaving as expected.

---

## Activating an Interface

If an interface is down, you can bring it up manually.

```text
sudo ifconfig eth0 up
```

or:

```text
sudo ip link set eth0 up
```

---

## Assigning an IP Address

To manually assign an IP address to an interface:

```text
sudo ifconfig eth0 192.168.1.2
```

This is useful in labs, isolated environments, or temporary setups.

---

## Setting a Netmask

```text
sudo ifconfig eth0 netmask 255.255.255.0
```

The netmask defines which part of the address represents the network and which part identifies the host.

---

## Configuring the Default Gateway

The default gateway is used to reach networks outside your local subnet.

```text
sudo route add default gw 192.168.1.1 eth0
```

If the gateway is wrong or missing, you may reach local systems but nothing beyond them.

---

## Configuring DNS Resolution

DNS translates domain names into IP addresses. When DNS is broken, almost everything feels broken.

Temporary DNS changes can be made by editing:

```text
/etc/resolv.conf
```

Example:

```text
nameserver 8.8.8.8
nameserver 8.8.4.4
```

⚠️ Changes here are usually **not persistent** and may be overwritten by NetworkManager or `systemd-resolved`.

---

## Making Network Settings Persistent

To ensure settings survive reboots, you must edit the appropriate configuration files.

On older Ubuntu-based systems:

```text
sudo vim /etc/network/interfaces
```

Example static configuration:

```text
auto eth0
iface eth0 inet static
  address 192.168.1.2
  netmask 255.255.255.0
  gateway 192.168.1.1
  dns-nameservers 8.8.8.8 8.8.4.4
```

After saving the file, restart networking:

```text
sudo systemctl restart networking
```

---

## Access Control Models in Practice

### Discretionary Access Control (DAC)

With DAC, the **owner of a resource** controls access. Linux file permissions are the classic example. This model is flexible but easy to misuse.

---

### Mandatory Access Control (MAC)

MAC enforces access rules based on **security labels**, not user choice. A process can only access resources that match its clearance level.

MAC is common in:

* Government environments
* Military systems
* Financial and healthcare infrastructure

It is restrictive but highly secure.

---

### Role-Based Access Control (RBAC)

RBAC assigns permissions based on **roles**, not individual users. This simplifies management in large environments and reduces human error.

---

## Network Troubleshooting

You will troubleshoot network issues constantly during assessments.

Common tools include:

* `ping`
* `traceroute`
* `netstat`
* `tcpdump`
* `wireshark`
* `nmap`

### Testing Connectivity

```text
ping 8.8.8.8
```

This confirms whether the target is reachable and measures latency.

---

### Tracing Routes

```text
traceroute www.example.com
```

This shows the path packets take and helps identify dropped traffic, firewalls, or routing issues.

---

### Viewing Active Connections

```text
netstat -a
```

This displays listening services and established connections, which is extremely useful during enumeration.

---

## Common Network Issues

You will frequently encounter:

* No connectivity
* DNS resolution failures
* Packet loss
* Poor performance

Typical causes include:

* Misconfigured firewalls or routers
* Incorrect IP, gateway, or DNS settings
* Network congestion
* Faulty hardware
* Outdated or unpatched software

Being able to separate **configuration mistakes** from **security issues** is a key skill.

---

## Hardening Linux Networks

Linux provides several mechanisms to reduce attack surface:

### SELinux

* Kernel-level MAC system
* Very granular control
* Powerful but complex

### AppArmor

* Profile-based MAC system
* Easier to manage
* Common on Ubuntu

### TCP Wrappers

* IP-based access control
* Simple and effective
* Limited to network-level restrictions

Each tool has strengths and trade-offs, and real systems often use a combination.

---

## Practice and Exploration

You will not build network intuition by reading alone.

Use a personal virtual machine. Take snapshots. Change IPs. Break DNS. Restrict access. Monitor traffic. Fix what you break.

Explaining what you learn to others—through notes, discussions, or write-ups—will reinforce your understanding far more effectively than passive study.

This is how network confidence is built.
