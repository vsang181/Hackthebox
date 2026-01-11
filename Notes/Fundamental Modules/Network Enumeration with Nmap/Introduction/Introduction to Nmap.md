## Introduction to Nmap

Nmap (Network Mapper) is an open-source network analysis and security auditing tool. You will mainly use it to discover hosts on a network, identify open ports, enumerate running services, and gather basic operating system information. Internally, Nmap works by crafting and sending raw packets, then analysing how targets respond.

Nmap is written using a mix of C, C++, Python, and Lua. Beyond basic scanning, it can also help you understand how defensive controls such as firewalls, packet filters, and intrusion detection systems (IDS) behave when probed.

---

## Common Use Cases

You will regularly see Nmap used for the following tasks:

* Auditing the security posture of networks
* Supporting penetration testing and red-team simulations
* Verifying firewall and IDS behaviour
* Mapping networks and identifying reachable hosts
* Analysing service responses and banners
* Discovering open TCP and UDP ports
* Performing early-stage vulnerability discovery

In practice, Nmap is often the **first tool** used during enumeration.

---

## Nmap Architecture and Capabilities

Nmap supports multiple scanning phases that can be combined or run independently, depending on what you are trying to learn about the target.

At a high level, Nmap can perform:

* **Host discovery** – determine which systems are alive
* **Port scanning** – identify open, closed, or filtered ports
* **Service detection** – identify services and versions
* **Operating system detection** – fingerprint the underlying OS
* **Scripted interaction** – interact with services using the Nmap Scripting Engine (NSE)

Understanding which phase you are in helps you avoid noisy or unnecessary scans.

---

## Basic Syntax

Nmap follows a simple and flexible syntax:

```
nmap <scan-types> <options> <target>
```

You can stack multiple scan types and options together depending on your goal.

---

## Scan Techniques Overview

Nmap provides a wide range of scanning techniques, each using different packet structures and behaviours. You can view them using:

```
nmap --help
```

Some commonly encountered scan types include:

* **TCP SYN scan (`-sS`)**
* **TCP Connect scan (`-sT`)**
* **UDP scan (`-sU`)**
* **TCP Null, FIN, and Xmas scans (`-sN`, `-sF`, `-sX`)**
* **ACK and Window scans (`-sA`, `-sW`)**
* **Idle (zombie) scan (`-sI`)**
* **IP protocol scan (`-sO`)**

Each technique exists for a reason, often to bypass filtering, reduce noise, or infer firewall behaviour.

---

## TCP SYN Scan (`-sS`)

The TCP SYN scan is one of the most commonly used Nmap techniques and is the default when run with sufficient privileges.

Key characteristics:

* Sends a packet with the **SYN** flag set
* Does **not** complete the TCP three-way handshake
* Faster and less intrusive than a full connection scan
* Often referred to as a “half-open” scan

Response interpretation:

* **SYN-ACK received** → port is **open**
* **RST received** → port is **closed**
* **No response** → port is **filtered** (likely by a firewall)

Because no full connection is established, this scan is harder to log and is widely used during reconnaissance.

---

## Example: TCP SYN Scan

Command:

```
sudo nmap -sS localhost
```

Sample output:

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5432/tcp open  postgresql
5901/tcp open  vnc-1
```

How to read this output:

* **PORT** – the port number and protocol
* **STATE** – whether the port is open, closed, or filtered
* **SERVICE** – Nmap’s best guess at the running service

From this single scan, you already know which services are worth enumerating further and which attack surface is exposed.
