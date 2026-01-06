# Network Layer (OSI Layer 3)

The **Network Layer**, also known as **Layer 3** in the OSI model, is responsible for getting data packets from the source to the destination **across one or more networks**. At this layer, packets are no longer confined to a single local segment. Instead, they may need to pass through multiple intermediate systems before reaching their final target.

You should think of Layer 3 as the layer that answers a very simple but critical question:

**“Where does this packet need to go next?”**

---

## What the Network Layer Actually Does

Packets cannot always be delivered directly to the destination host. Very often, the sender and receiver are located in **different subnets** or even completely different networks. In such cases, packets must be forwarded through **routing nodes**, typically routers.

At this layer, the system is responsible for:

* Identifying network nodes using logical addresses
* Deciding the next hop for each packet
* Forwarding packets from node to node
* Managing routing paths across networks

Packets move hop by hop until they finally reach the destination network.

---

## Core Responsibilities of Layer 3

In practical terms, Layer 3 focuses on just two fundamental functions.

### Logical Addressing

Logical addressing allows every device to be uniquely identified **across networks**.

At this layer:

* Hosts are identified using IP addresses
* Networks are structured using subnetting and CIDR
* Address information is evaluated for every packet sent

Without logical addressing, there is no way to determine where a packet should be delivered.

---

### Routing

Routing is the process of selecting the **best path** for a packet to reach its destination.

This involves:

* Evaluating destination IP addresses
* Consulting routing tables
* Forwarding packets to the appropriate next-hop router

Each router only needs to know where to send the packet **next**, not the full end-to-end path.

---

## How Packet Forwarding Works

When a packet is sent:

1. The source host creates a packet with a destination IP address
2. If the destination is not in the local subnet, the packet is sent to a router
3. The router examines the destination address
4. The router forwards the packet to the next routing node
5. This process repeats until the packet reaches the destination network

Important detail:
When a router forwards a packet, it **does not process higher-layer data**. The packet is not passed to Layers 4–7. Instead, it is simply given a new intermediate destination and forwarded onward.

---

## Routing Across Different Networks

Layer 3 handles communication even when:

* The source and destination use different subnet structures
* Multiple routing domains are involved
* Packets must traverse wide-area networks

In all cases, the packet still travels through the **entire communication network**, being forwarded by intermediate routers until delivery is complete.

---

## Layer 3 Protocols You Should Know

Each OSI layer defines its own protocols. These protocols operate independently of the layers above and below them.

The most commonly used **Network Layer protocols** include:

* **IPv4 / IPv6** – Logical addressing and packet delivery
* **IPsec** – Secure IP communication
* **ICMP** – Error reporting and diagnostics
* **IGMP** – Multicast group management
* **RIP** – Distance-vector routing protocol
* **OSPF** – Link-state routing protocol

Some of these protocols extend beyond a single layer, but their primary responsibility still sits at Layer 3.

---

## Why This Layer Matters to You

From a security and testing perspective, Layer 3 is where:

* Network segmentation is enforced
* Routing mistakes create blind spots
* Misconfigured subnets break visibility
* Pivoting paths either exist or do not

If you do not understand Layer 3 behaviour, you will:

* Miss reachable targets
* Misinterpret scan results
* Fail to route traffic correctly during exploitation

---

## Key Takeaways

* Layer 3 enables communication **between networks**
* Logical addressing tells packets *where* to go
* Routing decides *how* they get there
* Routers forward packets without inspecting higher-layer data
* Most network-level security boundaries live here

Once you are comfortable with the Network Layer, routing diagrams and subnet layouts stop being confusing and start becoming powerful tools you can reason about quickly and confidently.
