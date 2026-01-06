# Wireless Networks

Wireless networks allow devices to communicate **without physical cabling**, using radio frequency (RF) signals instead. You will encounter them everywhere: homes, offices, campuses, cafés, and enterprise environments. From a security perspective, wireless networking is critical because **anyone within range can potentially interact with the network**.

At a high level, wireless networking removes the need for cables but introduces new challenges around **authentication, encryption, interference, and signal leakage** 

---

## How Wireless Communication Works

Every device on a wireless network has a **wireless adapter**. This adapter converts data into RF signals, transmits them over the air, and receives RF signals from other devices.

Key points you should keep in mind:

* Communication happens over **radio waves**
* Signals travel through walls and open space
* Anyone within range can capture those signals
* Distance, obstacles, and noise affect reliability

Wireless networks can operate over different ranges depending on the technology:

* **WLAN (WiFi)**: homes and offices (short range)
* **WWAN (cellular: 3G / 4G / 5G)**: city or regional coverage

---

## Wireless Access Points (WAPs)

A **Wireless Access Point (WAP)** acts as the central coordinator of a WiFi network. It connects wireless clients to a wired network and controls who is allowed to transmit.

When your device wants to send data:

1. It first requests permission from the WAP
2. The WAP grants or denies access
3. Approved data is transmitted as RF signals
4. Receiving devices convert RF signals back into data

The WAP is therefore a **critical control point** and a high-value target during wireless assessments.

---

## WiFi Frequencies and Signal Behaviour

Most WiFi networks operate on:

* **2.4 GHz**
* **5 GHz**

Signal strength and reliability are influenced by:

* Transmit power
* Physical obstacles (walls, furniture, metal)
* RF noise from other devices
* Channel congestion

To remain usable, WiFi relies on techniques such as:

* Error correction
* Spread spectrum transmission

---

## Connecting to a WiFi Network

To join a wireless network, your device must know:

* The **SSID** (network name)
* The **security settings**
* Any required credentials (password or certificates)

WiFi uses the **IEEE 802.11** standard to define how devices associate with a network.

### Association Request Information

When a device attempts to connect, it sends an association request containing information such as:

* MAC address
* SSID
* Supported data rates
* Supported channels
* Supported security protocols (e.g. WPA2, WPA3)

Once accepted, the device can communicate with:

* The WAP
* Other wireless clients
* The Internet via the wired network

---

## Hidden SSIDs

An SSID can be hidden by disabling broadcast beacons. This prevents casual discovery but **does not provide real security**.

Important point:

* Hidden SSIDs still appear in authentication traffic
* Anyone capturing packets can identify the SSID

Security through obscurity does not work.

---

## WEP Challenge-Response Handshake

WEP uses a **challenge-response authentication process**:

| Step | Actor  | What Happens                        |
| ---- | ------ | ----------------------------------- |
| 1    | Client | Sends association request           |
| 2    | WAP    | Sends challenge string              |
| 3    | Client | Encrypts challenge using shared key |
| 4    | WAP    | Verifies response and authenticates |

To detect transmission errors, WEP uses **CRC (Cyclic Redundancy Check)**.

### Why CRC Breaks WEP

CRC is calculated on **plaintext**, not ciphertext. Because the CRC value is transmitted alongside encrypted data, attackers can infer plaintext content and manipulate packets without knowing the key.

This design flaw is one of the reasons **WEP is fundamentally broken**.

---

## Wireless Security Features

Wireless networks rely on multiple layers of protection:

### Encryption

Protects confidentiality of transmitted data.

Common schemes:

* WEP (obsolete)
* WPA2
* WPA3

### Access Control

Determines who is allowed to join the network, often using:

* Passwords
* MAC address filtering
* Certificate-based authentication

### Firewall

Wireless routers often include firewalls that:

* Block unsolicited inbound traffic
* Reduce exposure to external attacks

---

## Encryption Protocols

### WEP

WEP uses the **RC4 cipher** and shared keys.

Two variants exist:

| Protocol        | IV     | Secret Key |
| --------------- | ------ | ---------- |
| WEP-40 / WEP-64 | 24-bit | 40-bit     |
| WEP-104         | 24-bit | 80-bit     |

Problems you should remember:

* Small IV space
* Reused keys
* Weak cryptography
* Easy to brute force

**WEP should be considered insecure in all cases.**

---

### WPA / WPA2 / WPA3

WPA replaced WEP and introduced stronger security.

Key improvements:

* Stronger encryption (AES)
* Improved key management
* Better authentication mechanisms

Types you will encounter:

* **WPA-Personal** (PSK)
* **WPA-Enterprise** (central authentication)

In enterprise and critical environments, **WPA2 or WPA3 is the minimum acceptable standard**.

---

## Authentication Protocols

### EAP Framework

Extensible Authentication Protocol (EAP) provides a framework for wireless authentication.

Two important variants:

#### LEAP

* Uses shared secrets
* Vulnerable to credential compromise
* Considered insecure

#### PEAP

* Uses TLS tunnelling
* Protects authentication exchange
* Much more resistant to interception

---

## TACACS+ in Wireless Environments

When a WAP authenticates users via **TACACS+**, authentication traffic is typically encrypted to protect credentials and session data.

Benefits:

* Centralised authentication
* Encrypted communication
* Reduced exposure of sensitive information

Encryption may use:

* TLS
* IPsec

---

## Disassociation Attacks

A **disassociation attack** forces clients to disconnect by sending forged disassociation frames.

Why this matters:

* Causes denial of service
* Forces clients to reconnect
* Can be used to stage MITM attacks

This attack can be launched:

* From inside the network
* From outside the network (within RF range)

---

## Wireless Hardening Techniques

You should always think defensively as well.

Effective hardening measures include:

* Disabling SSID broadcast (minor benefit)
* Enforcing WPA2/WPA3
* Using enterprise authentication
* Avoiding legacy protocols
* Monitoring wireless traffic

### MAC Filtering

Limits access by MAC address, but:

* MACs are trivial to spoof
* Provides minimal real security

### Deploying EAP-TLS

One of the strongest approaches:

* Certificate-based authentication
* Mutual trust verification
* Resistant to credential theft

---

## Key Takeaways

* Wireless networks expose traffic over RF
* Anyone in range can capture packets
* WEP is broken and must not be used
* WPA2/WPA3 are the modern baseline
* Hidden SSIDs do not equal security
* Disassociation attacks enable further exploitation
* Enterprise wireless security requires strong authentication

If you treat wireless networks as “just WiFi”, you will miss **a massive attack surface**. Understanding how wireless works at a protocol level is essential for both attacking and defending modern environments.
