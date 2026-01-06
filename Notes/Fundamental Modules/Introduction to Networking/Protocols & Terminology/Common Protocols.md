# Common Protocols

Internet communication works because everyone agrees to follow the same **rules**. These rules are defined as **protocols**, standardised in **RFCs**, and they describe *how* data is structured, transmitted, received, and interpreted. Without protocols, networks would simply not function.

Whenever two devices communicate, they:

* Establish a communication channel (wired or wireless)
* Agree on a transport mechanism
* Exchange data according to well-defined protocol rules

The two most important transport mechanisms you will work with are **TCP** and **UDP**. Everything else builds on top of them 

---

## Transmission Control Protocol (TCP)

**TCP** is a **connection-oriented** protocol. Before any real data is sent, both sides establish a virtual connection using the **three-way handshake**. This connection remains active until communication is complete.

You should think of TCP as:

* Reliable
* Ordered
* Error-checked
* Slower due to overhead

### How TCP Is Used in Practice

When you enter a URL in your browser:

1. Your browser establishes a TCP connection to the web server
2. An HTTP request is sent over that connection
3. The server responds with HTML, CSS, JavaScript, and other resources
4. The connection is closed once the transfer is complete

TCP ensures:

* All packets arrive
* Packets are reassembled in the correct order
* Lost packets are retransmitted

This reliability is why TCP is used for services where **accuracy matters more than speed**.

---

### Common TCP-Based Protocols

| Protocol                              | Acronym    | Port         | What It Is Used For                |
| ------------------------------------- | ---------- | ------------ | ---------------------------------- |
| Telnet                                | Telnet     | 23           | Legacy remote login (plaintext)    |
| Secure Shell                          | SSH        | 22           | Secure remote login and tunnelling |
| Simple Network Management Protocol    | SNMP       | 161–162      | Network monitoring and management  |
| Hypertext Transfer Protocol           | HTTP       | 80           | Web traffic (plaintext)            |
| Hypertext Transfer Protocol Secure    | HTTPS      | 443          | Encrypted web traffic              |
| Domain Name System                    | DNS        | 53           | Name resolution                    |
| File Transfer Protocol                | FTP        | 20–21        | File transfers (plaintext)         |
| Trivial File Transfer Protocol        | TFTP       | 69           | Lightweight file transfer          |
| Network Time Protocol                 | NTP        | 123          | Time synchronisation               |
| Simple Mail Transfer Protocol         | SMTP       | 25           | Sending email                      |
| Post Office Protocol                  | POP3       | 110          | Retrieving email                   |
| Internet Message Access Protocol      | IMAP       | 143          | Accessing email                    |
| Server Message Block                  | SMB        | 445          | Windows file and resource sharing  |
| Network File System                   | NFS        | 111, 2049    | Unix/Linux file sharing            |
| Kerberos                              | Kerberos   | 88           | Authentication and authorisation   |
| Lightweight Directory Access Protocol | LDAP       | 389          | Directory services                 |
| Remote Desktop Protocol               | RDP        | 3389         | Remote desktop access              |
| Remote Procedure Call                 | RPC        | 135, 137–139 | Remote procedure execution         |
| Secure Copy                           | SCP        | 22           | Secure file copy                   |
| Microsoft SQL Server                  | ms-sql-s   | 1433         | Microsoft SQL database access      |
| Oracle TNS Listener                   | oracle-tns | 1521/1526    | Oracle database connections        |
| Squid Proxy                           | http-proxy | 3128         | HTTP proxy and caching             |

---

## User Datagram Protocol (UDP)

**UDP** is a **connectionless** protocol. There is no handshake and no guarantee that packets arrive, arrive in order, or arrive at all.

You should think of UDP as:

* Fast
* Lightweight
* Unreliable by design
* Stateless

### How UDP Is Used in Practice

When you stream a video or join a live call:

* Some packet loss is acceptable
* Speed is more important than reliability
* Missing packets are not retransmitted

This makes UDP ideal for **real-time communication**.

---

### Common UDP-Based Protocols

| Protocol                            | Acronym    | Port | What It Is Used For     |
| ----------------------------------- | ---------- | ---- | ----------------------- |
| Domain Name System                  | DNS        | 53   | Name resolution         |
| Trivial File Transfer Protocol      | TFTP       | 69   | File transfer           |
| Network Time Protocol               | NTP        | 123  | Time synchronisation    |
| Simple Network Management Protocol  | SNMP       | 161  | Device monitoring       |
| Routing Information Protocol        | RIP        | 520  | Routing updates         |
| Internet Key Exchange               | IKE        | 500  | VPN key exchange        |
| Bootstrap Protocol                  | BOOTP      | 68   | Host bootstrapping      |
| Dynamic Host Configuration Protocol | DHCP       | 67   | IP assignment           |
| NetBIOS Name Service                | netbios-ns | 137  | Windows name resolution |
| Microsoft SQL Browser               | ms-sql-m   | 1434 | SQL Server discovery    |
| Universal Plug and Play             | UPnP       | 1900 | Device discovery        |
| PostgreSQL                          | PGSQL      | 5432 | Database access         |
| Virtual Network Computing           | VNC        | 5900 | Remote desktop          |
| Syslog                              | SYSLOG     | 514  | Log forwarding          |
| Internet Relay Chat                 | IRC        | 194  | Real-time chat          |
| X Display Manager Control Protocol  | XDMCP      | 177  | Remote graphical login  |

---

## ICMP (Internet Control Message Protocol)

**ICMP** is not used to transfer application data. Instead, it is used for **error reporting and diagnostics**.

You encounter ICMP constantly, even if you do not realise it.

### Common ICMP Uses

* Testing reachability (`ping`)
* Tracing network paths (`traceroute`)
* Reporting delivery failures
* Signalling timeouts and routing issues

---

### ICMP Request Types

| Request              | Purpose                    |
| -------------------- | -------------------------- |
| Echo Request         | Tests reachability         |
| Timestamp Request    | Queries remote system time |
| Address Mask Request | Requests subnet mask       |

---

### ICMP Message Types

| Message                 | Meaning                       |
| ----------------------- | ----------------------------- |
| Echo Reply              | Response to echo request      |
| Destination Unreachable | Packet could not be delivered |
| Redirect                | Suggests a better route       |
| Time Exceeded           | TTL expired                   |
| Parameter Problem       | Header issue                  |
| Source Quench           | Flow control (deprecated)     |

---

### Time-To-Live (TTL)

Every IP packet includes a **TTL value** that limits how long it can exist.

* Each router decrements TTL by 1
* When TTL reaches 0, the packet is discarded
* An ICMP *Time Exceeded* message is sent back

TTL is useful for:

* Estimating hop distance
* Tracing routes
* Fingerprinting operating systems (heuristic only)

Typical default TTL values:

* Windows: 128
* Linux / macOS: 64
* Solaris: 255

These values can be changed, so they are **not definitive**.

---

## VoIP (Voice over IP)

**VoIP** allows voice and multimedia communication over IP networks instead of traditional phone lines.

Common examples include:

* Skype
* WhatsApp
* Zoom
* Slack
* Google Meet

---

### Common VoIP Protocols and Ports

* **SIP**: TCP/5060, TCP/5061
* **H.323**: TCP/1720 (legacy, less common)

---

### SIP (Session Initiation Protocol)

SIP is a **signalling protocol** used to:

* Initiate sessions
* Maintain sessions
* Modify sessions
* Terminate sessions

#### Common SIP Methods

| Method   | Purpose                 |
| -------- | ----------------------- |
| INVITE   | Start a session         |
| ACK      | Acknowledge INVITE      |
| BYE      | End a session           |
| CANCEL   | Cancel a pending INVITE |
| REGISTER | Register a user agent   |
| OPTIONS  | Query capabilities      |

---

### Information Disclosure via SIP

SIP can unintentionally expose useful information.

For example:

* Enumerating valid users
* Discovering supported codecs
* Identifying reachable endpoints

The **OPTIONS** method is commonly abused for:

* User enumeration
* Capability discovery
* Reconnaissance prior to brute-force attacks

In enterprise environments, you may encounter files such as:

```text
SEPxxxx.cnf
```

These are configuration files used by **Cisco Unified Communications Manager** and may contain:

* Device identifiers
* Firmware versions
* Network configuration
* Authentication-related settings

---

## Why These Protocols Matter to You

As a security practitioner, protocols are not abstract concepts.

They determine:

* How you scan
* What ports you target
* Which attacks are possible
* Where information leaks occur
* How traffic can be intercepted or abused

If you understand **how and why a protocol exists**, exploiting or defending it becomes far more straightforward.

---

## Key Takeaways

* Protocols define all network communication
* TCP prioritises reliability, UDP prioritises speed
* ICMP is critical for diagnostics and reconnaissance
* TTL can reveal routing behaviour and hints about systems
* VoIP and SIP often leak information
* Knowing protocols is more important than memorising port numbers

As you move forward, these protocols will stop being names on a list and start becoming **tools you recognise immediately in traffic, scans, and configurations**.
