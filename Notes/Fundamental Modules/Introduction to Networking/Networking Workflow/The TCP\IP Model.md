# The TCP/IP Model

The **TCP/IP model**, often called the **Internet Protocol Suite**, is the practical networking model that powers the Internet and most modern networks. Like OSI, it is a **layered reference model**, but it is far more closely aligned with how networks actually operate in the real world.

The name *TCP/IP* comes from two core protocols:

* **Transmission Control Protocol (TCP)**
* **Internet Protocol (IP)**

In OSI terms:

* **IP** maps to the **Network layer (Layer 3)**
* **TCP** maps to the **Transport layer (Layer 4)**

Despite the name, TCP/IP represents an **entire family of protocols**, not just these two.

---

## TCP/IP Layers and Responsibilities

The TCP/IP model uses **four layers**, combining several OSI layers into broader functional groups.

| Layer | Name        | What You Should Think About                                                |
| ----- | ----------- | -------------------------------------------------------------------------- |
| 4     | Application | Application-level protocols and services used by software to exchange data |
| 3     | Transport   | Session-oriented (TCP) and connectionless (UDP) communication              |
| 2     | Internet    | Logical addressing, packet creation, and routing                           |
| 1     | Link        | Transmission of packets over the physical network medium                   |

The reduced number of layers makes TCP/IP simpler and more flexible than OSI, without losing functionality.

---

## How TCP/IP Works in Practice

With TCP/IP, any application can communicate over **any compatible network**, regardless of where the destination host is located.

At a high level:

* **IP** ensures packets reach the correct destination network
* **TCP** manages reliable data transfer between applications
* **UDP** provides fast, connectionless communication where reliability is handled elsewhere
* **Ports** tie network traffic to specific applications

This separation allows networks to scale globally without requiring applications to understand routing or physical transmission details.

---

## OSI vs TCP/IP (Conceptual Comparison)

<img width="2576" height="1533" alt="net_models4" src="https://github.com/user-attachments/assets/c5f383b5-0c36-4a23-800b-675368dae756" />

| OSI Model    | TCP/IP Model |
| ------------ | ------------ |
| Application  | Application  |
| Presentation | Application  |
| Session      | Application  |
| Transport    | Transport    |
| Network      | Internet     |
| Data Link    | Link         |
| Physical     | Link         |

Key takeaway:

* **OSI** is granular and descriptive
* **TCP/IP** is compact and implementation-focused

---

## Core Responsibilities Within TCP/IP

The TCP/IP protocol suite is responsible for several critical tasks. You should understand *which protocol does what*.

### Logical Addressing — IP

**Protocol:** IP

Because networks contain many hosts across different locations, traffic must be logically structured.

IP handles:

* Network and host addressing
* Subnetting
* CIDR-based routing
* Ensuring packets reach the correct network

Without IP, there is no scalable network communication.

---

### Routing — IP

**Protocol:** IP

Each router on the path decides where to forward a packet next. The sender does not need to know the full route—only the destination.

This hop-by-hop decision-making allows:

* Dynamic routing
* Fault tolerance
* Global scalability

---

### Error Handling and Flow Control — TCP

**Protocol:** TCP

TCP establishes a **virtual connection** between sender and receiver.

It provides:

* Acknowledgements
* Retransmission of lost packets
* Flow control
* Congestion handling

This makes TCP reliable, but slower than UDP.

---

### Application Support — TCP / UDP

**Protocols:** TCP and UDP

Ports provide a **software abstraction layer** that allows multiple applications to communicate simultaneously on the same host.

For example:

* Web servers
* Mail servers
* Database services

All can coexist because ports uniquely identify each service.

---

### Name Resolution — DNS

**Protocol:** DNS

Humans use names, networks use numbers.

DNS translates:

* **Fully Qualified Domain Names (FQDNs)**
* Into **IP addresses**

Without DNS, you would need to remember IP addresses for every service you access.

---

## Why This Model Matters to You

As a penetration tester or security professional, you constantly rely on the TCP/IP model:

* Scans operate at the Internet and Transport layers
* Exploits target Application-layer services
* Firewalls filter traffic between layers
* Packet captures expose behaviour across layers

When something fails, TCP/IP helps you narrow it down quickly:

* Can I reach the host? (Internet layer)
* Is the port open? (Transport layer)
* Is the service responding correctly? (Application layer)

---

## Key Takeaways

* TCP/IP is the **real-world networking model**
* It combines OSI layers for practicality
* IP handles addressing and routing
* TCP ensures reliable communication
* UDP prioritises speed over reliability
* DNS bridges human-friendly names and IP addresses

Once you are comfortable with TCP/IP, network behaviour stops feeling mysterious and starts feeling **predictable and logical**.
