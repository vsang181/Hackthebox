# Firewall Setup on Linux

Firewalls exist to **control, inspect, and restrict network traffic** between different network segments, such as internal and external networks or separate security zones. Their primary purpose is to reduce exposure by allowing only authorised traffic while blocking or monitoring anything that does not explicitly belong 

On Linux systems, firewalling is not an afterthought. It is deeply integrated into the operating system and is one of the most important defensive controls you will encounter, both when hardening systems and when assessing them.

As a penetration tester, you should always ask yourself two questions:

* What traffic is allowed?
* What traffic *should not* be allowed but currently is?

---

## Firewalls in the Linux Ecosystem

Linux provides built-in firewall capabilities through the **Netfilter framework**, which operates at the kernel level. Netfilter exposes hooks that allow packets to be inspected, modified, accepted, or dropped as they move through the networking stack.

Several user-space tools exist to configure these rules:

* **iptables** – the traditional and long-standing firewall interface
* **nftables** – the modern replacement with improved syntax and performance
* **ufw (Uncomplicated Firewall)** – a simplified frontend for iptables
* **firewalld** – a dynamic, zone-based firewall manager

Although newer tools exist, **iptables is still widely deployed** and frequently encountered in real environments, especially on older systems and servers.

---

## iptables: A Brief History

iptables was introduced with the Linux 2.4 kernel in the year 2000, replacing older tools such as `ipchains` and `ipfwadm`. It quickly became the standard firewall solution due to its flexibility, performance, and tight kernel integration.

With iptables, administrators can filter traffic based on:

* Source and destination IP addresses
* Ports and protocols
* Connection state
* Packet contents and rate limits

This makes it suitable for defending against common threats such as port scans, denial-of-service attacks, and unauthorised access attempts.

---

## How iptables Is Structured

To work effectively with iptables, you must understand its internal structure. iptables rules are organised using five core concepts:

| Component   | Purpose                                                  |
| ----------- | -------------------------------------------------------- |
| **Tables**  | Categorise rules based on packet handling purpose        |
| **Chains**  | Ordered lists of rules applied to specific traffic paths |
| **Rules**   | Individual conditions and actions                        |
| **Matches** | Criteria used to identify packets                        |
| **Targets** | Actions applied when a rule matches                      |

Each packet passing through the system is evaluated against these components in a defined order.

---

## Tables in iptables

Tables group rules by **what kind of processing they perform**. Each table has a specific role.

| Table    | Description                 | Built-in Chains                                 |
| -------- | --------------------------- | ----------------------------------------------- |
| `filter` | Standard packet filtering   | INPUT, OUTPUT, FORWARD                          |
| `nat`    | Network address translation | PREROUTING, POSTROUTING                         |
| `mangle` | Packet header modification  | PREROUTING, OUTPUT, INPUT, FORWARD, POSTROUTING |
| `raw`    | Special packet handling     | PREROUTING, OUTPUT                              |

You will spend most of your time working with the **filter** table during basic firewall configuration.

---

## Chains Explained

Chains are **ordered rule lists** that define how packets are processed depending on their direction.

### Built-in Chains

Each table contains predefined chains. For example, the `filter` table includes:

* **INPUT** – traffic destined for the local system
* **OUTPUT** – traffic originating from the local system
* **FORWARD** – traffic passing through the system

The `nat` table focuses on address translation:

* **PREROUTING** – before routing decisions
* **POSTROUTING** – after routing decisions

---

### User-Defined Chains

You can create custom chains to **organise rules logically**. This improves readability and maintainability.

For example, instead of repeating the same rules for multiple web servers, you can:

* Create a custom chain for HTTP traffic
* Forward matching packets into that chain

This modular approach becomes essential as rulesets grow.

---

## Rules and Targets

A rule defines **what to match** and **what to do** when a match occurs.

Targets specify the action to take.

| Target       | Action                                |
| ------------ | ------------------------------------- |
| `ACCEPT`     | Allow the packet                      |
| `DROP`       | Silently discard the packet           |
| `REJECT`     | Drop the packet and notify the sender |
| `LOG`        | Log packet details                    |
| `SNAT`       | Modify source IP address              |
| `DNAT`       | Modify destination IP address         |
| `MASQUERADE` | Dynamic source NAT                    |
| `REDIRECT`   | Redirect traffic to another port      |
| `MARK`       | Tag packets for advanced routing      |

---

## Creating a Basic Rule

To allow incoming SSH traffic on port 22:

```text
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

This rule:

* Appends (`-A`) to the INPUT chain
* Matches TCP traffic
* Checks destination port 22
* Accepts matching packets

---

## Matches: Defining the Conditions

Matches specify **which packets a rule applies to**.

| Match          | Description               |
| -------------- | ------------------------- |
| `-p`           | Protocol (tcp, udp, icmp) |
| `--dport`      | Destination port          |
| `--sport`      | Source port               |
| `-s`           | Source IP address         |
| `-d`           | Destination IP address    |
| `-m state`     | Connection state          |
| `-m conntrack` | Connection tracking       |
| `-m limit`     | Rate limiting             |
| `-m string`    | Payload string matching   |
| `-m mac`       | MAC address matching      |

Example: allow HTTP traffic on port 80:

```text
sudo iptables -A INPUT -p tcp -m tcp --dport 80 -j ACCEPT
```

---

## Typical Practical Exercises

To build real understanding, you should practise tasks such as:

1. Launch a web server on TCP port 8080 and block incoming access
2. Modify rules to allow traffic on TCP/8080
3. Block traffic from a specific IP address
4. Allow traffic from a trusted IP address
5. Block traffic based on protocol
6. Allow traffic based on protocol
7. Create a custom user-defined chain
8. Forward traffic to that chain
9. Delete a specific rule
10. List all existing rules

These exercises force you to think about **packet flow**, not just commands.

---

## Important Notes for Assessments

From an offensive perspective, firewalls often fail due to:

* Overly permissive rules
* Forgotten exceptions
* Misunderstanding of chain order
* Inconsistent rules across tables

Always enumerate:

* Active rules
* Default policies
* Exposed ports
* NAT behaviour

Firewalls are powerful, but **misconfiguration is common**.

---

## Key Takeaway

iptables is not just a defensive tool. It is a **map of trust assumptions**. If you can read and understand a ruleset, you gain insight into how a system expects traffic to behave—and where those expectations can be broken.

Spend time experimenting. Break rules. Lock yourself out. Fix it. This is how firewall logic becomes intuitive.
