# Networking Models

When systems communicate over a network, the data does not just magically move from one host to another. The process follows **well-defined models** that describe *how* data is prepared, transferred, and understood. The two models you must be comfortable with are:

* **OSI (ISO/OSI) model**
* **TCP/IP model**

Think of these models as **mental frameworks**, not strict implementations. You will use them to reason about problems, analyse traffic, and understand where things break.

---

## Why Networking Models Exist

Raw bits travelling over a cable are not very useful on their own. Networking models help you mentally transform those bits into **meaningful data**, step by step.

Each model divides communication into **layers**, where:

* Each layer has a specific responsibility
* Each layer only interacts with the layer above and below it
* Complexity is reduced by separation of concerns

---

## The OSI Model

The **OSI (Open Systems Interconnection) model** is a **reference model** designed to describe network communication in a structured and systematic way.

<img width="2576" height="1533" alt="net_models4_updated" src="https://github.com/user-attachments/assets/c88490a3-e569-4698-8e75-7e07558dbd35" />

It was published by:

* The **International Organization for Standardization (ISO)**
* The **International Telecommunication Union (ITU)**

You will often hear it called the **ISO/OSI model**.

### OSI Layers (Top to Bottom)

| Layer | Name         | What You Should Think About                |
| ----- | ------------ | ------------------------------------------ |
| 7     | Application  | User-facing services (HTTP, FTP, SMTP)     |
| 6     | Presentation | Encoding, encryption, compression          |
| 5     | Session      | Session setup, management, teardown        |
| 4     | Transport    | Reliability, ports, flow control (TCP/UDP) |
| 3     | Network      | Logical addressing and routing (IP)        |
| 2     | Data Link    | Frames, MAC addresses, switching           |
| 1     | Physical     | Bits, signals, cables, voltages            |

The OSI model is **strict and descriptive**. It is not how networks are implemented in practice, but it is extremely useful for **analysis and troubleshooting**.

---

## The TCP/IP Model

The **TCP/IP model** is the model that actually runs the Internet.

Despite the name, it does **not** refer only to:

* Transmission Control Protocol (TCP)
* Internet Protocol (IP)

Instead, TCP/IP represents an **entire protocol family**, including:

* TCP
* UDP
* IP
* ICMP
* ARP
* Many others

The TCP/IP model is more **practical and flexible** than OSI.

### TCP/IP Layers

| Layer       | Purpose                                      |
| ----------- | -------------------------------------------- |
| Application | Application protocols (HTTP, DNS, FTP, SMTP) |
| Transport   | End-to-end communication (TCP, UDP)          |
| Internet    | Routing and logical addressing (IP, ICMP)    |
| Link        | Physical and data link operations            |

---

## OSI vs TCP/IP (Side-by-Side)

<img width="2576" height="1533" alt="net_models_pdu2_updated" src="https://github.com/user-attachments/assets/8af50839-cd1c-4f0d-806e-17a95f7df819" />

| OSI Model    | TCP/IP Model |
| ------------ | ------------ |
| Application  | Application  |
| Presentation | Application  |
| Session      | Application  |
| Transport    | Transport    |
| Network      | Internet     |
| Data Link    | Link         |
| Physical     | Link         |

Key difference:

* **OSI** separates responsibilities more finely
* **TCP/IP** groups responsibilities based on real-world implementation

---

## Which One Should You Use?

The answer is: **both**, but for different reasons.

* Use **TCP/IP** to understand how connections actually work
* Use **OSI** to break problems down and analyse them layer by layer

As a penetration tester, you will constantly switch between these perspectives without realising it.

---

## Packet Transfers and PDUs

In layered models, each layer communicates using a **Protocol Data Unit (PDU)**.

Each layer wraps the data it receives from the layer above by adding its own header. This process is called **encapsulation**.

### Common PDUs by Layer

| Layer                                | PDU Name                       |
| ------------------------------------ | ------------------------------ |
| Application / Presentation / Session | Data                           |
| Transport                            | Segment (TCP) / Datagram (UDP) |
| Network                              | Packet                         |
| Data Link                            | Frame                          |
| Physical                             | Bits                           |

---

## Encapsulation Explained (Step by Step)

When you browse a website:

1. **Application layer** creates the request data
2. **Transport layer** adds port information (TCP or UDP)
3. **Network layer** adds IP addressing
4. **Data Link layer** adds MAC addressing
5. **Physical layer** converts everything into bits and transmits it

On the receiving side, the process happens in **reverse**:

* Each layer removes its header
* Passes the remaining data upward
* Until the application finally uses the data

<img width="2650" height="959" alt="packet_transfer" src="https://github.com/user-attachments/assets/89a3dec2-81cf-4dc7-91eb-cdf8f952682b" />
---


## Why This Matters to You

When you:

* Capture packets
* Intercept traffic
* Analyse protocols
* Debug broken connections

You are mentally walking **up and down these layers**.

For example:

* A service is running, but the port is blocked → Layer 4 issue
* The IP is unreachable → Layer 3 issue
* The cable is unplugged → Layer 1 issue
* The request is malformed → Layer 7 issue

Without this mental model, troubleshooting becomes guesswork.

---

## Final Notes for You

* OSI is a **teaching and analysis model**
* TCP/IP is an **implementation model**
* Encapsulation explains how raw bits become usable data
* Layered thinking helps you isolate problems quickly

As you move into **network traffic analysis**, protocol abuse, and exploitation, these models will stop feeling abstract and start feeling **practical and essential**.
