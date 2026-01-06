# Virtual Private Networks (VPNs)

A **Virtual Private Network (VPN)** creates a **secure, encrypted tunnel** between a remote device and a private network over the public Internet. From your perspective as a user or administrator, a VPN makes it appear as if your system is physically connected to the internal network, even when you are working remotely 

This is commonly used when access to internal resources (such as servers, databases, or management interfaces) is restricted to the local network only.

---

## Why VPNs Exist

Imagine you are an administrator working from another city. Internal servers are intentionally unreachable from the Internet for security reasons. Without a VPN, you would have no direct way to manage them.

With a VPN:

1. You authenticate to a VPN server over the Internet
2. An encrypted tunnel is established
3. Your device is assigned an **internal IP address**
4. You can now access internal systems as if you were on-site

This approach is widely used because it is:

* Secure
* Flexible
* Cost-effective
* Scalable

---

## Common VPN Use Cases

You will typically encounter VPNs in the following scenarios:

* Remote employee access (work from home or travel)
* Administrative access to internal servers
* Secure access to email, file servers, and internal applications
* Connecting multiple branch offices into a single logical network

Because VPNs use the public Internet rather than dedicated links, they are significantly cheaper than leased lines or private circuits.

---

## Core VPN Components

For a VPN to function, several components must work together:

| Component          | Purpose                                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------------------- |
| **VPN Client**     | Installed on the remote device and initiates the VPN connection (for example, OpenVPN or built-in OS clients). |
| **VPN Server**     | Accepts incoming VPN connections and routes traffic between clients and the internal network.                  |
| **Encryption**     | Protects confidentiality using algorithms such as AES and protocols like IPsec.                                |
| **Authentication** | Verifies identity using credentials, shared secrets, or certificates.                                          |

If any of these components are weak or misconfigured, the entire VPN becomes a liability.

---

## Common VPN Ports and Protocols

Depending on the VPN technology, different ports and protocols are used:

* **PPTP**: TCP/1723
* **IKEv1 / IKEv2**: UDP/500
* **IPsec NAT Traversal**: UDP/4500

At the IP layer, VPNs often rely on **Encapsulating Security Payload (ESP)** to encrypt and authenticate traffic.

---

## IPsec Overview

**IPsec (Internet Protocol Security)** is a widely used framework that provides **encryption, integrity, and authentication** at the IP layer.

Rather than protecting applications individually, IPsec secures **entire IP packets**, making it ideal for VPNs.

---

### IPsec Building Blocks

IPsec relies on two core protocols:

#### Authentication Header (AH)

* Provides integrity and authenticity
* Does **not** encrypt data
* Detects tampering

#### Encapsulating Security Payload (ESP)

* Encrypts the packet payload
* Optionally provides authentication
* Used in most modern VPN deployments

---

### IPsec Modes of Operation

IPsec can operate in two distinct modes:

| Mode               | Description                                                                                           |
| ------------------ | ----------------------------------------------------------------------------------------------------- |
| **Transport Mode** | Encrypts only the payload, leaving the original IP header intact. Common for host-to-host protection. |
| **Tunnel Mode**    | Encrypts the entire IP packet, including the header. Commonly used for VPN tunnels between networks.  |

For most enterprise VPNs, **tunnel mode** is the standard.

---

## Required Protocols for IPsec VPNs

When VPN traffic passes through a firewall, specific protocols must be allowed:

| Protocol                    | Port     | Purpose                                         |
| --------------------------- | -------- | ----------------------------------------------- |
| Internet Protocol           | IP/50–51 | Core IP routing foundation                      |
| Internet Key Exchange (IKE) | UDP/500  | Negotiates encryption keys using Diffie–Hellman |
| ESP (NAT-T)                 | UDP/4500 | Encrypts and authenticates VPN traffic          |

Blocking any of these will usually break the VPN connection.

---

## PPTP (Legacy VPN Protocol)

**Point-to-Point Tunnelling Protocol (PPTP)** is one of the earliest VPN technologies and is still supported by many operating systems.

However, you should treat PPTP as **cryptographically broken**.

Key issues:

* Uses **MS-CHAPv2** for authentication
* Relies on **DES encryption**
* Vulnerable to offline cracking
* Easily broken with modern hardware

Because of these weaknesses, PPTP has been largely replaced by:

* IPsec / IKEv2
* L2TP/IPsec
* OpenVPN
* WireGuard

You should never deploy PPTP in a modern environment, and encountering it during an assessment is a **serious red flag**.

---

## Security Perspective: Why VPNs Matter to You

From an offensive and defensive standpoint, VPNs are critical because:

* They often provide **direct internal network access**
* Compromised VPN credentials can bypass perimeter defences
* Weak authentication or outdated protocols lead to full network compromise
* VPN traffic is trusted by default once authenticated

A VPN is not just a tunnel. It is often the **front door to the internal network**.

---

## Key Takeaways

* VPNs create encrypted tunnels over the public Internet
* Remote users receive internal IP addresses
* IPsec is the most common enterprise VPN framework
* Tunnel mode encrypts the entire packet
* PPTP is insecure and obsolete
* VPN misconfigurations frequently lead to full network access

If you understand how VPNs work at a protocol level, you will be far better equipped to **attack, defend, and audit remote access infrastructure**.
