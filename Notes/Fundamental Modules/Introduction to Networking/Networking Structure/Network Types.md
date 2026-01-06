# Network Types

Every network is designed with a specific purpose in mind, which is why no two networks are truly identical. To make sense of this variety, networks are commonly grouped into **types** based on size, usage, and scope. When you first encounter these classifications, it can feel overwhelming, especially because some terms are rarely used outside textbooks.

To make this practical for you, this section is split into **commonly used terms** and **book terms**. You should be comfortable with the common terms in day-to-day work. The book terms are still worth knowing, but you are unlikely to hear them outside of exams, documentation, or very specific edge cases 

---

## Common Network Terminology

These are the network types you will actually encounter in real environments.

### Wide Area Network (WAN)

In everyday language, a **WAN** usually means *the Internet*. More accurately, a WAN is a network that connects **multiple LANs across large geographic areas**.

In practice, you will often see:

* A **WAN address**, which is reachable from outside
* A **LAN address**, which is used internally

A WAN does not have to be the public Internet. Large organisations and government bodies often operate **internal WANs**, sometimes referred to as:

* Intranets
* Air-gapped networks
* Private backbone networks

You can usually identify a WAN by:

* The use of WAN routing protocols such as **BGP**
* IP addresses that are **not** part of RFC 1918 private ranges:

  * `10.0.0.0/8`
  * `172.16.0.0/12`
  * `192.168.0.0/16`

---

### Local Area Network (LAN) and Wireless LAN (WLAN)

A **LAN** is a network limited to a relatively small physical area, such as:

* A home
* An office
* A single building
* A campus segment

A **WLAN** is functionally the same thing, except the connection medium is wireless instead of wired.

In most cases, LANs and WLANs use **private IP address ranges** defined in RFC 1918:

* `10.0.0.0/8`
* `172.16.0.0/12`
* `192.168.0.0/16`

Occasionally, you may encounter environments such as hotels or universities where devices are assigned **publicly routable IP addresses**. This is uncommon and usually driven by specific design choices.

From a security point of view, WLANs introduce additional risks because data is transmitted over the air rather than a cable, but logically they are still LANs.

---

### Virtual Private Network (VPN)

A **Virtual Private Network (VPN)** makes you feel as if your device is directly connected to a different network, even though the connection is carried over another network, usually the Internet.

There are three main VPN models you should understand.

---

#### Site-to-Site VPN

In a **site-to-site VPN**, both ends of the connection are network devices, typically routers or firewalls.

Key characteristics:

* Entire networks are connected together
* Used to link company offices across locations
* Devices communicate as if they were on the same local network

This is very common in enterprise environments.

---

#### Remote Access VPN

With a **remote access VPN**, your computer creates a **virtual network interface** that behaves as if it belongs to the remote network.

Important points:

* Traffic is routed through a virtual adapter (for example, a TUN interface)
* Access depends heavily on the routing table pushed by the VPN
* If only specific networks are routed, this is called **split tunnelling**

Split tunnelling is convenient for labs and learning platforms, but in corporate environments it is often discouraged. If malware is present on the client, traffic that bypasses the VPN may avoid network-based detection entirely.

---

#### SSL VPN

An **SSL VPN** operates through a web browser.

Instead of creating a full tunnel, it may:

* Stream applications
* Provide access to internal web services
* Offer full desktop sessions in the browser

This model has become increasingly popular as browsers have grown more capable. It reduces client-side setup but introduces different security considerations.

---

## Book Terms (Good to Know)

These terms are mostly used in documentation, academic material, and exams. You rarely hear them in daily operational conversations.

---

### Global Area Network (GAN)

A **GAN** refers to a worldwide network, such as the Internet.

However, global corporations may also operate **private GANs**, connecting internal WANs across continents using:

* Fibre-optic backbones
* Undersea cables
* Satellite links

The key idea is global scale, not public access.

---

### Metropolitan Area Network (MAN)

A **MAN** connects multiple LANs within a city or metropolitan area.

Common characteristics:

* High-speed fibre connections
* Leased lines
* Very low latency between sites

MANs are often used to connect:

* Company branches
* Data centres
* Government facilities within a region

From a performance perspective, communication inside a MAN can be almost as fast as within a single LAN.

---

### Personal Area Network (PAN) and Wireless PAN (WPAN)

A **PAN** is a very small network designed for short-range communication between personal devices.

Examples include:

* Connecting a phone to a laptop
* Tethering
* Peripheral communication

A **WPAN** is the wireless variant and commonly uses technologies such as:

* Bluetooth
* Wireless USB

A Bluetooth-based WPAN is often called a **piconet**.

These networks usually span only a few metres and are not suitable for connecting rooms or buildings.

In the context of **IoT**, WPANs are widely used for smart home and automation systems. Protocols such as **Z-Wave**, **ZigBee**, and **Insteon** are designed specifically for low-power, low-data-rate communication.

---

## What You Should Take Away

As you move forward, focus on **function, not labels**.

Always ask yourself:

* How large is this network?
* Who can talk to whom?
* What protocols are in use?
* Where are the trust boundaries?

If you understand how and why a network is designed, the terminology becomes secondary.
