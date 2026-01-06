# Vendor-Specific Networking and VLANs (Cisco-Focused Notes)

This section is a **practical, note-style overview** of vendor-specific networking concepts you will commonly encounter, with a strong focus on **Cisco IOS** and **VLAN technologies**. Read this as if you are building intuition for real environments rather than memorising theory. Everything here shows up sooner or later in production networks and security assessments 

---

## Cisco IOS – What You Need to Know

**Cisco IOS** is the operating system used by many Cisco routers and switches. If you are working in enterprise networks, you will run into it frequently.

Cisco IOS provides the core features required to operate modern networks, including:

* IPv4 and IPv6 support
* Routing and switching functionality
* Quality of Service (QoS)
* Security controls (ACLs, authentication, encryption)
* Virtualisation features such as **VRF** and **VPLS**

You usually interact with Cisco IOS via:

* **Command Line Interface (CLI)** (most common)
* **Graphical interfaces** (less common, device-dependent)

---

## Cisco IOS Authentication and Password Types

Cisco IOS uses different passwords for different privilege levels. Understanding these matters during audits and incident response.

| Password Type       | Purpose                                                        |
| ------------------- | -------------------------------------------------------------- |
| **User password**   | Controls basic login access                                    |
| **Enable password** | Grants access to privileged (enable) mode                      |
| **Secret**          | Protects specific services or management access                |
| **Enable secret**   | Encrypted password for enable mode (preferred and more secure) |

When accessing a Cisco device remotely (via **Telnet** or **SSH**), a clear indicator is the response:

```text
User Access Verification
Password:
```

This banner is often enough to fingerprint Cisco IOS remotely.

---

## VLANs – Why They Exist

A **Virtual Local Area Network (VLAN)** allows you to split one physical switch into multiple **logical broadcast domains**.

Think of VLANs as **mini-switches inside a single switch**.

Why this matters:

* Broadcast traffic stays contained
* Departments are isolated
* Performance improves
* Security boundaries are enforced at Layer 2

Each VLAN:

* Is its own broadcast domain
* Requires its own **IP subnet**
* Does not forward broadcasts to other VLANs

---

## Example VLAN Design (Mental Model)

| Department | VLAN ID | Subnet         |
| ---------- | ------- | -------------- |
| Servers    | 10      | 192.168.1.0/24 |
| C-Level    | 20      | 192.168.2.0/24 |
| Finance    | 30      | 192.168.3.0/24 |
| HR         | 40      | 192.168.4.0/24 |
| Marketing  | 50      | 192.168.5.0/24 |
| Support    | 60      | 192.168.6.0/24 |

Traffic in VLAN 30 **never reaches VLAN 40** unless explicitly routed.

---

## VLAN ID Ranges (Cisco)

Cisco switches support VLAN IDs **1–4094**.

Important details:

* **VLAN 1**: default VLAN (should not be deleted or modified)
* **0 and 4095**: reserved
* **1–1005**: normal-range VLANs
* **1006–4094**: extended-range VLANs

Normal-range VLANs are stored in `vlan.dat` and can have:

* Name
* Type
* State
* MTU

---

## VLAN Membership Types

### Static VLANs (Preferred)

* Ports are manually assigned to VLANs
* More secure
* Predictable behaviour

### Dynamic VLANs

* VLAN assignment based on MAC address or protocol
* Uses services like **VMPS**
* Flexible but increases attack surface

From a security perspective, **static VLANs are safer**. MAC-based VLAN assignment can be bypassed with MAC spoofing tools.

---

## Access Ports vs Trunk Ports

Every switch port is one of these:

### Access Port

* Carries traffic for **one VLAN**
* End devices are unaware of VLANs
* Most user-facing ports

### Trunk Port

* Carries traffic for **multiple VLANs**
* Connects switches or switch-to-router links
* Uses VLAN tagging

Misconfigured trunk ports are a **common attack vector**.

---

## VLAN Tagging – 802.1Q

Standard Ethernet frames do not include VLAN information. **IEEE 802.1Q** solves this by inserting a VLAN tag into the Ethernet frame.

Key fields:

* **TPID**: always `0x8100`
* **TCI**, which includes:

  * **PCP** (priority)
  * **DEI** (drop eligible)
  * **VID** (VLAN ID – 12 bits)

This allows **4094 usable VLAN IDs**.

### VLAN Tagging Concepts

* **Tagging**: inserting VLAN information
* **Untagging**: removing VLAN information before delivery to access ports

---

## VLAN Hopping Attacks

VLANs improve security but are **not immune to abuse**.

### DTP-Based VLAN Hopping

* Exploits Cisco **Dynamic Trunking Protocol**
* Attacker pretends to be a switch
* Switch negotiates a trunk
* Attacker sees traffic from multiple VLANs

Tools like **Yersinia** automate this attack.

---

### Double-Tagging VLAN Hopping

This attack embeds **two VLAN tags** into one frame.

High-level flow:

1. Outer tag matches native VLAN
2. First switch strips the outer tag
3. Inner tag survives and is forwarded
4. Frame lands in a different VLAN

Requirements:

* Attacker must be on the **native VLAN**
* Native VLAN must be misconfigured or unchanged

This attack is subtle and often missed.

---

## VXLAN – Scaling VLANs

Traditional VLANs scale poorly in large environments.

**VXLAN (RFC 7348)** solves this by:

* Encapsulating Layer 2 traffic over Layer 3
* Using a **24-bit VXLAN Network Identifier (VNI)**
* Supporting **~16 million segments**

VXLAN is common in:

* Data centres
* Cloud environments
* Virtualised infrastructures

Each VXLAN segment behaves like a logical Layer 2 network.

---

## Cisco Discovery Protocol (CDP)

**CDP** is a Cisco Layer 2 protocol that shares device information with neighbours.

Information exposed includes:

* Device hostname
* IP address
* Interface names
* OS version
* Hardware platform

Example takeaway:
If CDP is enabled, passive sniffing can reveal **critical infrastructure details** without authentication.

---

## Spanning Tree Protocol (STP)

**STP** prevents loops in Layer 2 networks by:

* Electing a root bridge
* Blocking redundant paths
* Maintaining a loop-free topology

STP traffic leaks:

* Root bridge MAC
* Network topology
* Timing parameters

This information is valuable during reconnaissance.

---

## Why This Matters to You

From a security perspective:

* VLANs define trust boundaries
* Misconfigurations enable lateral movement
* CDP and STP leak infrastructure intelligence
* VLAN hopping bypasses segmentation
* VXLAN expands attack surface in cloud environments

You should always assume:

* VLANs reduce risk, not eliminate it
* Layer 2 is often under-monitored
* Vendor defaults are dangerous

---

## Key Takeaways

* Cisco IOS dominates enterprise networking
* VLANs create logical broadcast domains
* Access vs trunk ports matter
* 802.1Q tagging enables VLAN separation
* VLAN hopping is still a real threat
* VXLAN enables massive Layer 2 scalability
* CDP and STP leak valuable reconnaissance data

Treat VLANs as **control mechanisms**, not security guarantees. Understanding how they work internally gives you leverage whether you are defending or attacking a network.
