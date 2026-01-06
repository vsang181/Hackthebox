# MAC Addresses

Every device connected to a network interface has a **Media Access Control (MAC) address**. This is a **Layer 2 (Data Link)** identifier that uniquely represents the physical or virtual network interface you are communicating with.

A MAC address is:

* **48 bits long**
* **6 octets (bytes)**
* Represented in **hexadecimal**

You will encounter MAC addresses across different networking standards, including:

* **Ethernet (IEEE 802.3)**
* **WLAN (IEEE 802.11)**
* **Bluetooth (IEEE 802.15)** 

The key idea to keep in mind is this:
**MAC addresses identify interfaces, not people or hosts.**

---

## MAC Address Formats

You will see MAC addresses written in several equivalent formats:

```text
DE:AD:BE:EF:13:37
DE-AD-BE-EF-13-37
DEAD.BEEF.1337
```

All of these represent the same 48-bit value.

---

## Binary and Hex Representation

Each MAC address consists of **six octets**, where each octet is 8 bits.

| Octet | Binary   | Hex |
| ----- | -------- | --- |
| 1     | 11011110 | DE  |
| 2     | 10101101 | AD  |
| 3     | 10111110 | BE  |
| 4     | 11101111 | EF  |
| 5     | 00010011 | 13  |
| 6     | 00110111 | 37  |

This low-level representation becomes important when analysing traffic or identifying special address flags.

---

## MAC Address Structure

A MAC address is split into **two logical halves**.

### OUI (Organisationally Unique Identifier)

* **First 3 bytes (24 bits)**
* Assigned by the **IEEE**
* Identifies the **manufacturer**

This allows you to fingerprint vendors directly from traffic captures.

---

### NIC / Device Identifier

* **Last 3 bytes (24 bits)**
* Assigned by the manufacturer
* Intended to be unique per interface

Together, these two parts ensure global uniqueness.

---

## How MAC Addresses Are Used in Communication

Whenever an IP packet is delivered on a local network, it must be wrapped inside a **Layer 2 frame**.

* If the destination is **in the same subnet**, the frame is sent directly to the destination MAC address
* If the destination is **in a different subnet**, the frame is sent to the **router’s MAC address (default gateway)**

This decision is handled automatically using **ARP**.

---

## Address Resolution Protocol (ARP)

ARP is used to map a **Layer 3 IP address** to a **Layer 2 MAC address**.

The process is simple:

1. A device broadcasts an **ARP request**
2. The host with the matching IP responds with an **ARP reply**
3. The sender caches the result and communicates directly

Example ARP exchange:

```text
Who has 10.129.12.101? Tell 10.129.12.100
10.129.12.101 is at AA:AA:AA:AA:AA:AA
```

This mapping is cached locally to avoid repeated broadcasts.

---

## Reserved and Special MAC Addresses

### Broadcast Address

```text
FF:FF:FF:FF:FF:FF
```

* Sent to **all devices** on the local network
* Commonly used by ARP and DHCP

---

### Multicast Addresses

Multicast MAC addresses deliver traffic to **multiple interested hosts**.

Example:

```text
01:00:5E:xx:xx:xx
```

* Traffic is sent once
* Hosts decide whether to accept it

---

### Locally Administered MAC Addresses

Certain MAC ranges are reserved for **local assignment**, often used by:

* Virtual machines
* Containers
* MAC spoofing

Examples:

```text
02:00:00:00:00:00
06:00:00:00:00:00
0A:00:00:00:00:00
0E:00:00:00:00:00
```

These are not globally registered with the IEEE.

---

## MAC Address Flags (First Octet)

The **first octet** contains two important control bits.

### Unicast vs Multicast

* **Last bit = 0** → Unicast (one-to-one)
* **Last bit = 1** → Multicast (one-to-many)

---

### Global vs Local

* **Second-last bit = 0** → Globally assigned (IEEE OUI)
* **Second-last bit = 1** → Locally administered

This is critical when identifying spoofed or virtualised interfaces.

---

## MAC Address Attack Vectors

MAC addresses should **never** be treated as a security control.

Common attack techniques include:

* **MAC spoofing**
  Changing your MAC address to impersonate another device

* **MAC flooding**
  Overloading a switch’s CAM table to force broadcast behaviour

* **MAC filtering bypass**
  Gaining access by cloning an allowed MAC address

Any security design relying solely on MAC addresses is weak by definition.

---

## ARP Spoofing (ARP Poisoning)

ARP has **no authentication**, which makes it vulnerable.

In an ARP spoofing attack:

* You send forged ARP replies
* Victims associate your MAC address with a legitimate IP (often the gateway)
* Traffic is redirected through your system

Example poisoning behaviour:

```text
10.129.12.255 is at CC:CC:CC:CC:CC:CC
```

This allows:

* Traffic interception
* Credential theft
* Man-in-the-middle attacks

Tools commonly used include Ettercap and Cain & Abel.

---

## Defensive Considerations

ARP-based attacks are mitigated through:

* Encrypted protocols (TLS, IPsec)
* Network segmentation
* Static ARP entries (limited use)
* IDS/IPS monitoring

As an attacker or defender, you should always assume **Layer 2 is untrusted**.

---

## Key Takeaways

* MAC addresses identify **network interfaces**, not hosts
* They operate only within the **local network**
* ARP bridges IP and MAC addressing
* MAC addresses are easy to spoof
* ARP is unauthenticated and exploitable
* Layer 2 attacks are often invisible to poorly monitored networks

If you understand MAC addressing and ARP properly, many “mysterious” network behaviours suddenly make complete sense.
