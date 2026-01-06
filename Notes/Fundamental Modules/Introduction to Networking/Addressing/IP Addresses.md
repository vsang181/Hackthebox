# IP Addresses

Every device on a network must be uniquely identifiable. At the most basic level, devices inside a single local network can identify each other using **Media Access Control (MAC) addresses**. This works well **within one network**, but it breaks down as soon as communication needs to happen **between different networks**.

To solve this, the Internet relies on **IP addressing**, specifically **IPv4** and **IPv6**. An IP address allows data to be routed across multiple networks and ensures that packets arrive at the correct destination, regardless of where that destination is located 

---

## MAC Address vs IP Address (Mental Model)

A useful way to think about this is with a postal analogy:

* **IPv4 / IPv6**
  The street address and postcode of a building (where the building is located)

* **MAC address**
  The specific flat or apartment inside that building

MAC addresses work locally. IP addresses work globally.

---

## IPv4 Structure

IPv4 is still the most widely used addressing scheme.

An **IPv4 address**:

* Is **32 bits long**
* Is divided into **4 octets**
* Each octet contains **8 bits**
* Each octet ranges from **0 to 255**

IPv4 addresses are written in **dotted-decimal notation**.

### Example

| Representation | Value                                 |
| -------------- | ------------------------------------- |
| Binary         | `01111111.00000000.00000000.00000001` |
| Decimal        | `127.0.0.1`                           |

Each network interface (NIC, printer, router, virtual interface) must have a **unique IP address within its network**.

---

## Address Space and Ownership

IPv4 provides:

* **4,294,967,296** total addresses

The address space is divided into:

* **Network portion**
* **Host portion**

In home networks, the **router** typically assigns host addresses.
On the public Internet, address allocation is coordinated by **IANA** and regional registries.

---

## Legacy IPv4 Address Classes (Historical Context)

Historically, IPv4 addresses were divided into fixed classes. You still see these referenced, even though they are largely obsolete.

| Class | CIDR      | Address Range               | Hosts per Network |
| ----- | --------- | --------------------------- | ----------------- |
| A     | /8        | 1.0.0.0 – 127.255.255.255   | ~16 million       |
| B     | /16       | 128.0.0.0 – 191.255.255.255 | ~65,000           |
| C     | /24       | 192.0.0.0 – 223.255.255.255 | 254               |
| D     | Multicast | 224.0.0.0 – 239.255.255.255 | N/A               |
| E     | Reserved  | 240.0.0.0 – 255.255.255.255 | N/A               |

These classes were replaced because they were inflexible and wasteful.

---

## Subnet Masks

To divide networks into smaller, more manageable pieces, we use **subnetting**.

A **subnet mask**:

* Is also 32 bits long
* Uses `1` bits to represent the network portion
* Uses `0` bits to represent the host portion

### Example

```text
IPv4 address: 192.168.10.39
Subnet mask:  255.255.255.0
```

Binary representation of the subnet mask:

```text
11111111.11111111.11111111.00000000
```

This means:

* First 24 bits → network
* Last 8 bits → hosts

---

## Network, Broadcast, and Gateway Addresses

Every IPv4 subnet reserves **two addresses**:

* **Network address**
  Identifies the subnet itself

* **Broadcast address**
  Used to send traffic to all hosts in the subnet

In addition, most networks define a **default gateway**:

* The IP address of the router
* Used to reach other networks
* Commonly the **first or last usable address** in the subnet (convention, not a requirement)

---

## Broadcast Address

A **broadcast** sends a packet to **every device** in the local subnet.

Key properties:

* No response required
* Used for discovery and announcements
* Always the **last IP address** in the subnet

Example for `/24`:

```text
Network:   192.168.10.0
Broadcast: 192.168.10.255
```

---

## Binary Representation of IPv4

Understanding binary is essential for subnetting and routing.

Each octet uses the following bit values:

| Bit Value | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |
| --------- | --- | -- | -- | -- | - | - | - | - |

### Example: 192.168.10.39

| Octet | Binary   | Calculation    | Result |
| ----- | -------- | -------------- | ------ |
| 1st   | 11000000 | 128 + 64       | 192    |
| 2nd   | 10101000 | 128 + 32 + 8   | 168    |
| 3rd   | 00001010 | 8 + 2          | 10     |
| 4th   | 00100111 | 32 + 4 + 2 + 1 | 39     |

Complete representation:

```text
11000000.10101000.00001010.00100111
192.168.10.39
```

Subnet masks are calculated in the same way.

---

## CIDR (Classless Inter-Domain Routing)

**CIDR** replaces fixed address classes with a flexible system.

Instead of classes, CIDR uses a **suffix** that indicates how many bits belong to the network.

### Example

```text
192.168.10.39/24
```

This means:

* First **24 bits** → network
* Remaining **8 bits** → hosts

CIDR is simply a shorthand for the subnet mask:

```text
/24 = 255.255.255.0
```

---

## Why This Matters to You

As a security practitioner, IP addressing affects everything you do:

* Scanning scope
* Reachability of targets
* Pivoting paths
* Firewall rules
* Network segmentation
* Missed assets due to wrong assumptions

If you misunderstand subnetting, you will:

* Scan the wrong range
* Miss critical systems
* Misinterpret routing behaviour

---

## Key Takeaways

* MAC addresses work locally, IP addresses work globally
* IPv4 uses 32-bit addressing split into octets
* Subnet masks define network vs host portions
* Network and broadcast addresses are always reserved
* CIDR replaces rigid address classes
* Binary understanding makes subnetting predictable

Once IP addressing clicks, network layouts stop looking chaotic and start looking **structured and intentional**.
