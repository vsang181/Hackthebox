# Networking Key Terminology

The field of information technology is vast, and networking alone is large enough that you could spend an entire career specialising in just one or two areas. There are countless protocols, standards, tools, edge cases, and failure modes. You do **not** need to know everything.

What you *do* need is a **working vocabulary**.

This section is your **baseline glossary**. Think of it as the alphabet you must recognise before you can read, reason about, and discuss networking topics in later modules. The list below focuses on **commonly encountered protocols and technologies**. It is intentionally not exhaustive, and that is by design.

You should revisit and expand this list over time as you encounter new protocols in real environments 

---

## Core Networking and Security Protocols

| Protocol                           | Acronym | What You Should Know                                                               |
| ---------------------------------- | ------- | ---------------------------------------------------------------------------------- |
| Wired Equivalent Privacy           | WEP     | Legacy wireless security protocol; fundamentally broken and insecure.              |
| Secure Shell                       | SSH     | Encrypted remote access protocol used for command execution and tunnelling.        |
| File Transfer Protocol             | FTP     | Plaintext file transfer protocol; credentials are sent unencrypted.                |
| Simple Mail Transfer Protocol      | SMTP    | Used to send emails between servers and clients.                                   |
| Hypertext Transfer Protocol        | HTTP    | Application-layer protocol for web communication (plaintext).                      |
| Server Message Block               | SMB     | File, printer, and resource sharing protocol used heavily in Windows environments. |
| Network File System                | NFS     | Unix/Linux-based file sharing protocol.                                            |
| Simple Network Management Protocol | SNMP    | Used to monitor and manage network devices; often misconfigured.                   |
| Wi-Fi Protected Access             | WPA     | Wireless security protocol designed to replace WEP.                                |
| Temporal Key Integrity Protocol    | TKIP    | Legacy wireless encryption mechanism; weaker than modern alternatives.             |
| Network Time Protocol              | NTP     | Synchronises system clocks across a network.                                       |
| Virtual Local Area Network         | VLAN    | Logical network segmentation at Layer 2.                                           |
| VLAN Trunking Protocol             | VTP     | Cisco protocol for propagating VLAN information between switches.                  |

---

## Routing and Network Control Protocols

| Protocol                                   | Acronym | What You Should Know                                                               |
| ------------------------------------------ | ------- | ---------------------------------------------------------------------------------- |
| Routing Information Protocol               | RIP     | Distance-vector routing protocol; simple but inefficient.                          |
| Open Shortest Path First                   | OSPF    | Link-state routing protocol used in enterprise networks.                           |
| Interior Gateway Routing Protocol          | IGRP    | Legacy Cisco proprietary routing protocol.                                         |
| Enhanced Interior Gateway Routing Protocol | EIGRP   | Advanced Cisco routing protocol combining distance-vector and link-state concepts. |
| Spanning Tree Protocol                     | STP     | Prevents loops in Layer 2 Ethernet networks.                                       |
| Hot Standby Router Protocol                | HSRP    | Cisco protocol providing gateway redundancy.                                       |
| Virtual Router Redundancy Protocol         | VRRP    | Open standard for router redundancy.                                               |

---

## Authentication, Encryption, and Identity

| Protocol                                         | Acronym | What You Should Know                                                     |
| ------------------------------------------------ | ------- | ------------------------------------------------------------------------ |
| Pretty Good Privacy                              | PGP     | Encryption system for emails and files.                                  |
| Terminal Access Controller Access-Control System | TACACS  | Centralised authentication, authorisation, and accounting.               |
| Extensible Authentication Protocol               | EAP     | Authentication framework supporting multiple methods.                    |
| Lightweight EAP                                  | LEAP    | Cisco proprietary wireless authentication; insecure by modern standards. |
| Protected EAP                                    | PEAP    | Encapsulates EAP within an encrypted tunnel.                             |
| Internet Protocol Security                       | IPsec   | Provides encryption and authentication at the IP layer.                  |
| Internet Key Exchange                            | IKE     | Negotiates cryptographic keys for IPsec tunnels.                         |

---

## Voice, Messaging, and Real-Time Communication

| Protocol                       | Acronym | What You Should Know                                  |
| ------------------------------ | ------- | ----------------------------------------------------- |
| Session Initiation Protocol    | SIP     | Controls setup and teardown of voice and video calls. |
| Voice over IP                  | VoIP    | Technology enabling voice calls over IP networks.     |
| Network News Transfer Protocol | NNTP    | Used for distributing messages in newsgroups.         |

---

## Tunnelling and Remote Access

| Protocol                           | Acronym | What You Should Know                         |
| ---------------------------------- | ------- | -------------------------------------------- |
| Virtual Private Network            | VPN     | Encrypted tunnel over untrusted networks.    |
| Point-to-Point Tunnelling Protocol | PPTP    | Legacy VPN protocol; cryptographically weak. |
| Generic Routing Encapsulation      | GRE     | Encapsulation protocol often used with VPNs. |
| Remote Shell                       | RSH     | Legacy remote command execution; insecure.   |

---

## Web and Application Technologies

| Term                            | Acronym | What You Should Know                                       |
| ------------------------------- | ------- | ---------------------------------------------------------- |
| Asynchronous JavaScript and XML | AJAX    | Enables dynamic web content using JavaScript and XML/JSON. |
| Internet Server API             | ISAPI   | Microsoft framework for extending web servers.             |
| Uniform Resource Identifier     | URI     | Generic identifier for resources on the Internet.          |
| Uniform Resource Locator        | URL     | A specific type of URI identifying a resource location.    |
| Carriage Return Line Feed       | CRLF    | Line-ending characters; relevant for injection attacks.    |

---

## Enterprise and Industrial Systems

| Technology                               | Acronym | What You Should Know                                        |
| ---------------------------------------- | ------- | ----------------------------------------------------------- |
| Systems Management Server                | SMS     | Microsoft system management solution (legacy).              |
| Microsoft Baseline Security Analyzer     | MBSA    | Legacy vulnerability assessment tool from Microsoft.        |
| Supervisory Control and Data Acquisition | SCADA   | Industrial control systems used in critical infrastructure. |

---

## Network Infrastructure Concepts

| Term                        | Acronym | What You Should Know                            |
| --------------------------- | ------- | ----------------------------------------------- |
| Network Address Translation | NAT     | Translates private IP addresses to public ones. |
| Cisco Discovery Protocol    | CDP     | Cisco proprietary device discovery protocol.    |

---

## How You Should Use This List

You are **not expected to memorise everything** here immediately.

Instead:

* Recognise the names when you see them
* Understand their general purpose
* Know which layer they roughly operate on
* Be aware of which ones are legacy or insecure

As you progress:

* Revisit this list
* Add notes
* Cross-reference with real traffic, configs, and labs

Once these terms stop looking foreign, reading documentation and analysing network behaviour becomes **significantly easier**.
