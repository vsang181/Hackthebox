# TCP and UDP Connections

At the transport layer, you mainly deal with **TCP** and **UDP**. These two protocols define *how* data is delivered between hosts, and understanding their behaviour is essential when you analyse traffic, scan networks, or exploit services 

You will see both everywhere, but they are designed for very different goals.

---

## TCP vs UDP (Big Picture)

You can think of TCP and UDP as two different delivery philosophies.

### TCP (Transmission Control Protocol)

TCP is **connection-oriented** and prioritises **reliability**.

Key characteristics:

* A connection is established before data is sent
* Data delivery is guaranteed
* Lost packets are retransmitted
* Order of packets is preserved
* Slower, but reliable

A useful mental model is a **telephone call**:

* Both sides agree to communicate
* The call stays open until one side ends it
* If something is not heard correctly, it is repeated

Typical TCP use cases:

* Web traffic (HTTP / HTTPS)
* Email
* File transfers
* Authentication protocols

---

### UDP (User Datagram Protocol)

UDP is **connectionless** and prioritises **speed**.

Key characteristics:

* No handshake
* No delivery guarantees
* No retransmissions
* No ordering
* Faster, but unreliable

Think of UDP like **shouting information across a room**:

* You send the message
* You do not wait for confirmation
* If something is missed, it is simply lost

Typical UDP use cases:

* Video streaming
* Online gaming
* Voice over IP (VoIP)
* DNS queries

Speed matters more than perfect delivery in these scenarios.

---

## IP Packets (What Actually Travels on the Wire)

At Layer 3, data is carried inside **IP packets**. Every TCP segment or UDP datagram is wrapped inside an IP packet before being transmitted.

A simple analogy:

* The **IP packet** is the envelope
* The **payload** is the letter inside

---

## IP Packet Structure

An IP packet consists of:

* **Header** – routing and control information
* **Payload** – actual data (TCP, UDP, ICMP, etc.)

### IP Header Fields (What You Should Care About)

| Field               | Purpose                |
| ------------------- | ---------------------- |
| Version             | IPv4 or IPv6           |
| Header Length       | Size of the IP header  |
| Class of Service    | Traffic priority       |
| Total Length        | Full packet size       |
| Identification (ID) | Used for fragmentation |
| Flags               | Fragmentation control  |
| Fragment Offset     | Position of fragment   |
| Time To Live (TTL)  | Limits packet lifetime |
| Protocol            | TCP, UDP, ICMP, etc.   |
| Checksum            | Header integrity check |
| Source Address      | Sender IP              |
| Destination Address | Receiver IP            |
| Options             | Optional routing info  |

You will not memorise all of this, but you **must recognise it when analysing traffic**.

---

## IP ID Field and Host Fingerprinting

The **IP Identification (ID)** field is particularly interesting during traffic analysis.

* It is a 16-bit value (0–65535)
* Used to reassemble fragmented packets
* Often incremented sequentially

If you see packets from **different IP addresses** with **sequential IP IDs**, it strongly suggests:

* Multiple interfaces
* One physical host
* One operating system instance

This technique is frequently used in passive reconnaissance.

---

## Record Route (RR) Field

The **Record Route** option allows an IP packet to record the IP addresses of routers it passes through.

You can observe this using:

```bash
ping -R <target>
```

When enabled:

* Each router appends its IP address
* The return packet shows the full path

This is useful for:

* Network mapping
* Path discovery
* Identifying intermediate routing devices

Note: Many modern networks block or ignore this option.

---

## Traceroute (How Paths Are Discovered)

`traceroute` works by manipulating the **TTL** field.

High-level process:

1. Send a packet with TTL = 1
2. Router drops it and sends ICMP Time Exceeded
3. Increase TTL and repeat
4. Continue until the destination responds

Once the destination replies (SYN/ACK, RST, or ICMP), the path is complete.

This technique gives you:

* Router IP addresses
* Network topology insights
* Latency indicators

---

## TCP Internals (What Matters for Attacks)

TCP uses **segments**, each with its own header and payload.

Important TCP header fields:

* **Source Port** – sending application
* **Destination Port** – target application
* **Sequence Number** – packet ordering
* **Acknowledgement Number** – confirms receipt
* **Flags** – SYN, ACK, FIN, RST, etc.
* **Window Size** – flow control
* **Checksum** – integrity verification

From a security perspective, sequence numbers and flags are critical.

---

## UDP Internals

UDP is intentionally minimal.

UDP header contains:

* Source port
* Destination port
* Length
* Checksum

That is it.

There is:

* No handshake
* No state
* No confirmation

This simplicity makes UDP:

* Fast
* Stateless
* Easy to spoof

---

## Blind Spoofing

**Blind spoofing** is an attack where you send forged packets **without seeing the responses**.

Key idea:

* You fake source IPs, ports, and sequence numbers
* You guess or predict protocol behaviour
* You never observe replies directly

This is difficult but not impossible.

Blind spoofing can be used to:

* Disrupt connections
* Tear down sessions
* Inject traffic
* Interfere with routing or trust relationships

Historically, predictable TCP sequence numbers made this far easier. Modern systems mitigate this, but the concept still matters.

---

## Why This Matters to You

As a security practitioner, understanding TCP and UDP allows you to:

* Interpret packet captures correctly
* Detect spoofing and scanning activity
* Understand service behaviour
* Identify trust assumptions in protocols
* Abuse or defend against transport-layer weaknesses

If you only memorise ports and commands, you will miss what is *actually happening* on the wire.

---

## Key Takeaways

* TCP prioritises reliability; UDP prioritises speed
* IP packets carry all higher-layer data
* IP ID fields can leak host information
* Traceroute abuses TTL to map networks
* TCP state and sequencing matter for security
* UDP is fast, stateless, and easy to abuse
* Blind spoofing relies on guessing protocol state

Transport-layer knowledge turns packet noise into **meaningful signals**.
