## IPMI

Intelligent Platform Management Interface (IPMI) is a set of standardized specifications for out-of-band, hardware-level system management and monitoring. It operates as an autonomous subsystem and functions independently of the host BIOS, CPU, firmware, and operating system. This design allows administrators to manage and monitor systems even when they are powered off, unresponsive, or the operating system has failed.

IPMI communicates directly with the system hardware over a dedicated management interface and does not require OS-level authentication or shell access. It is commonly used for remote administration tasks that would otherwise require physical access to the server.

Typical IPMI use cases include:

* Managing or modifying BIOS and boot configuration before the operating system loads
* Powering on, powering off, or resetting a system that is fully powered down
* Accessing and recovering a host after a system crash or OS failure

Outside of recovery and power management scenarios, IPMI is also used for continuous hardware monitoring. This includes collecting metrics such as temperature, voltage levels, fan speeds, power supply status, and chassis intrusion events. It can query hardware inventory, review system event logs, and generate alerts, often integrated with monitoring platforms via SNMP.

Although the host system itself may be powered off, the IPMI subsystem requires standby power and an active LAN connection to function.

The IPMI specification was originally introduced by Intel in 1998 and is now supported by more than 200 hardware vendors, including Cisco, Dell, HPE, Supermicro, and Intel. IPMI version 2.0 introduced additional features such as Serial-over-LAN (SoL), allowing administrators to interact with the system’s serial console remotely.

To function correctly, IPMI relies on several core components:

* **Baseboard Management Controller (BMC):** A dedicated microcontroller responsible for implementing IPMI functionality
* **Intelligent Chassis Management Bus (ICMB):** Enables communication between multiple chassis
* **Intelligent Platform Management Bus (IPMB):** Internal bus extending BMC communication
* **IPMI memory:** Stores system event logs, sensor data records, and configuration information
* **Communication interfaces:** LAN, serial interfaces, ICMB, and PCI management buses

---

## Footprinting the Service

IPMI primarily communicates over UDP port **623** using RMCP/RMCP+. Systems exposing IPMI functionality are generally referred to as Baseboard Management Controllers (BMCs). In most enterprise environments, BMCs are implemented as embedded ARM-based systems running a stripped-down Linux OS and are directly integrated into the server motherboard. Some platforms also support add-on BMC cards via PCI.

Most enterprise-grade servers ship with a BMC or support adding one. The most commonly encountered implementations during internal penetration tests include:

* HPE iLO
* Dell DRAC / iDRAC
* Supermicro IPMI

Access to a BMC is extremely high impact. It effectively grants near-physical access to the system, allowing an attacker to monitor hardware, reboot or power off the host, mount virtual media, or reinstall the operating system. Many BMCs expose multiple management interfaces, including a web-based console, SSH or Telnet access, and the IPMI protocol over UDP/623.

A basic way to identify IPMI is by using Nmap’s `ipmi-version` NSE script:

```
sudo nmap -sU --script ipmi-version -p 623 <target>
```

This scan confirms whether IPMI is present and often identifies the supported protocol version.

An alternative approach is using Metasploit’s IPMI information discovery module:

```
use auxiliary/scanner/ipmi/ipmi_version
```

---

## Default Credentials

During internal assessments, it is common to find BMCs with default or weak credentials that were never changed after deployment. Common defaults to keep in mind include:

| Product         | Username      | Password                                    |
| --------------- | ------------- | ------------------------------------------- |
| Dell iDRAC      | root          | calvin                                      |
| HP iLO          | Administrator | Randomized 8-character (uppercase + digits) |
| Supermicro IPMI | ADMIN         | ADMIN                                       |

Testing default credentials should be standard practice for any newly discovered management interface. Successful authentication can immediately lead to full system compromise through the management console or remote shell access.

---

## Dangerous Settings and Protocol Weaknesses

If default credentials fail, IPMI 2.0 introduces a critical weakness in the RAKP authentication exchange. During the authentication process, the BMC sends a salted SHA1 or MD5 hash of the user’s password to the client before authentication is completed. This allows an attacker to retrieve password hashes for **any valid BMC user** without knowing the plaintext password.

These hashes can then be cracked offline using tools such as Hashcat (mode 7300). For example, factory-default HP iLO passwords can often be cracked using a constrained mask attack targeting uppercase letters and digits.

There is no complete fix for this issue, as it is inherent to the IPMI 2.0 specification. Mitigations include:

* Enforcing long, complex, non-reusable passwords
* Isolating BMC interfaces using strict network segmentation
* Restricting access to management networks via firewall rules and ACLs

IPMI should never be exposed to untrusted networks. It is frequently overlooked during internal penetration tests, despite being present in most enterprise environments. In multiple real-world assessments, cracked IPMI credentials were reused across other services, enabling SSH access to production servers and administrative access to monitoring platforms.

---

## Dumping IPMI Password Hashes

Metasploit provides a module capable of extracting IPMI password hashes using the RAKP weakness:

```
use auxiliary/scanner/ipmi/ipmi_dumphashes
```

Successful execution may return password hashes that can be cracked offline. In many cases, the hashes correspond to default credentials, resulting in immediate access to the BMC.

Once access is obtained, an attacker can log in to the management interface, escalate impact, and potentially leverage credential reuse to compromise additional systems. For this reason, IPMI enumeration and testing should be a mandatory step in any internal penetration testing methodology.
