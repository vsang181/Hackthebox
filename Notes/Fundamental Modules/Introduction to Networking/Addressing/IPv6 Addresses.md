# IPv6 Addresses

**IPv6** is the successor to IPv4 and was designed to solve long-standing limitations around address exhaustion, scalability, and end-to-end connectivity. Unlike IPv4, which uses 32-bit addresses, **IPv6 uses 128-bit addresses**, providing an effectively inexhaustible address space.

Address allocation for both IPv4 and IPv6 is coordinated by the **Internet Assigned Numbers Authority (IANA)**. While IPv4 is still widely used today, IPv6 is expected to become dominant over time. In practice, both protocols commonly run side by side using a **dual-stack** approach.

---

## Why IPv6 Exists

IPv6 follows the **end-to-end principle** much more closely than IPv4. Every interface can have a publicly routable address without relying on Network Address Translation (NAT).

Key design outcomes you should keep in mind:

* A single interface can hold **multiple IPv6 addresses**
* NAT is no longer a requirement
* Discovery and communication rely on **multicast**, not broadcast
* Addressing and routing scale far better than IPv4

---

## Advantages of IPv6 Over IPv4

IPv6 introduces several improvements that directly impact performance, management, and security:

* Vastly larger address space
* Stateless Address Autoconfiguration (**SLAAC**)
* Multiple IPv6 addresses per interface
* More efficient routing
* Built-in IPsec support
* Support for very large packet sizes (up to 4 GB using extension headers)

---

## IPv4 vs IPv6 at a Glance

| Feature            | IPv4            | IPv6             |
| ------------------ | --------------- | ---------------- |
| Address length     | 32-bit          | 128-bit          |
| OSI layer          | Network         | Network          |
| Address space      | ~4.3 billion    | ~340 undecillion |
| Representation     | Decimal         | Hexadecimal      |
| Prefix notation    | `10.10.10.0/24` | `fe80::1/64`     |
| Dynamic addressing | DHCP            | SLAAC / DHCPv6   |
| IPsec              | Optional        | Mandatory        |

---

## IPv6 Address Types

IPv6 defines **three address types**, each with a clear purpose:

| Type          | Description                                                   |
| ------------- | ------------------------------------------------------------- |
| **Unicast**   | Identifies a single interface                                 |
| **Anycast**   | Assigned to multiple interfaces; only one receives the packet |
| **Multicast** | Assigned to multiple interfaces; all receive the packet       |

> **Important:** IPv6 completely removes broadcast addresses. Multicast is used instead for discovery and group communication.

---

## Why Hexadecimal Is Used

A 128-bit binary value is not human-friendly. IPv6 therefore uses **hexadecimal notation**, which represents **16 possible values (0–F)** with a single character.

| Decimal | Hex | Binary |
| ------- | --- | ------ |
| 10      | A   | 1010   |
| 11      | B   | 1011   |
| 12      | C   | 1100   |
| 13      | D   | 1101   |
| 14      | E   | 1110   |
| 15      | F   | 1111   |

This allows large binary values to be expressed compactly and readably.

---

## Hexadecimal Example (IPv4 Comparison)

To understand hex encoding, look at how an IPv4 address appears in hexadecimal:

| Representation | 1st Octet | 2nd Octet | 3rd Octet | 4th Octet |
| -------------- | --------- | --------- | --------- | --------- |
| Binary         | 11000000  | 10101000  | 00001100  | 10100000  |
| Hex            | C0        | A8        | 0C        | A0        |
| Decimal        | 192       | 168       | 12        | 160       |

IPv6 uses the same idea, just on a much larger scale.

---

## IPv6 Address Structure

An IPv6 address consists of **128 bits (16 bytes)** and is written as:

* **8 blocks**
* Each block is **16 bits (4 hex digits)**
* Blocks are separated by colons (`:`)

### Full vs Shortened Form

```text
Full:  fe80:0000:0000:0000:dd80:b1a9:6687:2d3b/64
Short: fe80::dd80:b1a9:6687:2d3b/64
```

To keep addresses readable:

* Leading zeros in each block are removed
* One sequence of consecutive zero blocks may be replaced with `::`

---

## Network Prefix and Interface Identifier

An IPv6 address is split into two logical parts:

* **Network Prefix** (network or subnet)
* **Interface Identifier** (host portion)

By default:

* The prefix length is **/64**
* The remaining 64 bits identify the interface

Other common prefix lengths you will encounter include:

* `/32`
* `/48`
* `/56`

When using an ISP-assigned IPv6 range, you typically receive a **shorter prefix** (for example `/56`) and create `/64` subnets internally.

---

## IPv6 Address Notation Rules (RFC 5952)

IPv6 notation follows strict formatting rules:

* All hexadecimal characters are **lowercase**
* Leading zeros within a block are omitted
* One or more consecutive zero blocks may be replaced by `::`
* The `::` shortcut may be used **only once**, starting from the left

Following these rules ensures consistency and avoids ambiguity.

---

## Why IPv6 Matters to You

From a security perspective, IPv6 is often **enabled by default** and **poorly monitored**.

You should assume that:

* IPv6 traffic may bypass IPv4-only controls
* Hosts may have reachable IPv6 addresses even when IPv4 appears filtered
* Multicast and neighbour discovery expose additional attack surface

Ignoring IPv6 does not make it disappear.

---

## Key Takeaways

* IPv6 uses 128-bit hexadecimal addresses
* NAT is no longer required
* Interfaces can have multiple IPv6 addresses
* Broadcast is replaced by multicast
* `/64` is the standard subnet size
* Proper notation matters for clarity and tooling

Once you are comfortable reading IPv6 addresses, they stop looking intimidating and start looking **structured, predictable, and extremely powerful**.
