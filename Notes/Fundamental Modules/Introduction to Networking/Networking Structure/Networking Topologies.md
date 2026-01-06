# Networking Topologies

A **network topology** describes how devices in a network are arranged and how they are connected, either **physically** or **logically**. As you work with real networks, you will realise that topology is not just an academic concept. It directly affects **performance, reliability, scalability, and security**.

When you understand the topology in use, you can better predict how traffic flows, where bottlenecks may occur, and which components are critical points of failure 

---

## Physical vs Logical Topology

It is important to separate these two ideas in your head.

### Physical Topology

The **physical topology** describes how devices are physically connected:

* Cabling layout
* Fibre runs
* Wireless coverage
* Port-to-port connections

For wired networks, this includes:

* Coaxial cabling
* Twisted-pair cabling
* Fibre-optic cabling

For wireless networks, this includes:

* Wi-Fi
* Cellular
* Satellite

---

### Logical Topology

The **logical topology** describes how data actually moves across the network, regardless of the physical layout.

For example:

* Devices may be physically arranged in a circle
* Logically, they may still operate as a star topology

This distinction is critical. The physical layout you see does not always reflect how traffic is handled.

---

## Core Components of Network Topology

You can break topology design into three broad areas.

---

### 1. Connections (Transmission Media)

Networks rely on transmission media to carry signals.

**Wired**

* Coaxial cable
* Twisted-pair cable
* Fibre-optic cable

**Wireless**

* Wi-Fi
* Cellular
* Satellite

The choice of medium affects:

* Speed
* Distance
* Interference
* Security

---

### 2. Nodes

A **node** is any device that can send, receive, or forward data.

Examples include:

* Network Interface Controllers (NICs)
* Repeaters
* Hubs
* Bridges
* Switches
* Routers and modems
* Gateways
* Firewalls

Some nodes are full computers, while others are simple embedded devices with no general-purpose operating system.

---

### 3. Classification

Topologies are often described using abstract models. These models help you reason about behaviour but do not always match the real-world layout.

A network can be:

* **Physical** (actual cabling and devices)
* **Logical** (how traffic flows)

Most modern networks are **logical abstractions** layered on top of physical infrastructure.

---

## Common Network Topologies

Below are the fundamental topology types you should recognise. More complex networks are almost always **hybrids** of these basics.

---

## Point-to-Point

A **point-to-point** topology is the simplest possible network.

* A dedicated connection between two devices
* No intermediate hosts
* Direct communication

This model is common in:

* Traditional telephony
* Dedicated links between routers

Do not confuse this with **peer-to-peer (P2P) architectures**, which describe how services are organised, not how devices are physically connected.

<img width="2576" height="684" alt="topo_p2p" src="https://github.com/user-attachments/assets/19a451b6-cfa1-42e6-995e-b3e7cb05216b" />

---

## Bus

In a **bus topology**, all hosts share a single transmission medium.

Key characteristics:

* Every device can see all traffic
* Only one device can transmit at a time
* No central control component

Historically, this was common with coaxial Ethernet. Today, it is largely obsolete due to scalability and reliability issues.

<img width="2576" height="1294" alt="topo_bus" src="https://github.com/user-attachments/assets/ef7eb7b8-2f3d-4549-97c8-ea347a366d9f" />

---

## Star

A **star topology** connects all hosts to a central device.

Typically, this central component is:

* A switch
* A router
* (Historically) a hub

All traffic passes through the central node, which makes it:

* Easy to manage
* Easy to expand
* A potential single point of failure

Most modern Ethernet LANs are logically star-based.

<img width="2576" height="1707" alt="topo_star" src="https://github.com/user-attachments/assets/aaedf3c7-db30-43cf-af92-1ee13baaadb0" />

---

## Ring

In a **ring topology**, each device connects to exactly two others:

* One incoming connection
* One outgoing connection

Data travels in a fixed direction around the ring, often controlled by a **token** mechanism. Only the device holding the token may transmit.

Many modern “ring” networks are actually **logical rings** built on physical star topologies.

<img width="2576" height="1746" alt="topo_ring" src="https://github.com/user-attachments/assets/ab070271-f788-4381-a65c-1e437d1d7242" />

---

## Mesh

A **mesh topology** focuses on redundancy and resilience.

There are two variants:

### Fully Meshed

* Every node connects to every other node
* Extremely resilient
* High cost and complexity

### Partially Meshed

* Only critical nodes have multiple connections
* Endpoints may have a single link

Mesh topologies are common in:

* WANs
* MANs
* Core routing infrastructure

They allow traffic to be rerouted automatically if a node fails.

<img width="2576" height="2024" alt="topo_mesh" src="https://github.com/user-attachments/assets/f831c167-f903-49e8-a0b7-c64bcc032687" />

---

## Tree

A **tree topology** is essentially an extended star.

Characteristics:

* Hierarchical structure
* Multiple star networks connected together
* Clear parent–child relationships

This topology is widely used in:

* Enterprise LANs
* Campus networks
* Metropolitan networks (MANs)

Modern structured cabling and spanning-tree designs naturally form tree-like structures.

<img width="2576" height="1715" alt="topo_tree" src="https://github.com/user-attachments/assets/6be94806-bb78-4de3-bf18-a655c6e9b875" />

---

## Hybrid

A **hybrid topology** combines two or more different topologies.

For example:

* Star networks connected via a bus
* Mesh core with star-based access networks

Most real-world networks are hybrids, even if they are described using a single dominant topology.

<img width="2576" height="1726" alt="topo_hybrid" src="https://github.com/user-attachments/assets/90bf738c-be2a-4e64-8f47-c69852e752a9" />

---

## Daisy Chain

In a **daisy chain**, devices are connected sequentially, one after another.

Key points:

* Forms a linear chain
* A failure in the middle can disrupt everything downstream
* Common in industrial and automation systems (for example, CAN bus)

This topology is based on physical arrangement rather than logical control mechanisms.

<img width="2576" height="1716" alt="topo_daisy-chain" src="https://github.com/user-attachments/assets/fdb21fe9-2a65-44ff-83c7-ee7bfe3ab3e2" />

---

## What You Should Take Away

When you look at a network, do not focus only on labels.

Instead, ask yourself:

* How does traffic actually flow?
* Where are the single points of failure?
* Which nodes are most critical?
* How easy would it be to isolate or pivot through this design?

Once you can answer those questions, network topology stops being theory and starts becoming a powerful analytical tool.
