# SNMP

Simple Network Management Protocol (SNMP) is used to **monitor and manage network devices** such as routers, switches, servers, printers, and IoT devices. Beyond monitoring, SNMP can also be used to **change configuration values remotely**, depending on how it is configured.

* **SNMP queries** typically use **UDP/161**
* **SNMP traps** (unsolicited event notifications) typically use **UDP/162**

SNMP is still common because it is lightweight and widely supported, but it is also frequently misconfigured, which makes it valuable during footprinting.

---

## Core Concepts

### Agents and Managers

* **Agent:** runs on the device being monitored (exposes management data).
* **Manager:** the monitoring system or client (queries the agent and receives traps).

### MIB

A **Management Information Base (MIB)** is essentially a “dictionary” describing what can be queried and how it is structured.

* Stored as text files (ASN.1-based).
* MIBs do not store the live values; they define **where** the values live and how to interpret them.

### OID

An **Object Identifier (OID)** is the unique numeric address for a single value in the SNMP tree.

* Written in dotted notation (for example: `.1.3.6.1.2.1.1.1.0`)
* Longer OIDs generally mean more specific information.

---

## SNMP Versions

### SNMPv1

* Oldest and still seen in small/legacy environments.
* **No encryption**
* Weak security model (easy to intercept and misuse)

### SNMPv2c

* “Community-based” (the “c”).
* Adds functionality, but security is still essentially the same as v1.
* **Community strings are sent in plain text**, so sniffing works if you can see the traffic.

### SNMPv3

* Stronger security model:

  * Authentication (user-based)
  * Encryption (privacy)
* More secure but also more complex to deploy, which is why many environments still have v2c.

---

## Community Strings

Community strings are effectively **password-like tokens** for SNMPv1/v2c.

Common patterns you will see:

* `public` (read-only, extremely common)
* `private` (sometimes read-write)

Because they are often weak and transmitted unencrypted, community strings are a frequent weak point.

---

## Default Configuration Example (snmpd)

Typical `/etc/snmp/snmpd.conf` content shows:

* Contact and location metadata
* Which OID subtrees are exposed (`view`)
* Community strings and permissions (`rocommunity`, `rwcommunity`)
* Whether the daemon is only listening locally or on a network interface (`agentaddress`)

A common “safe-ish” pattern is exposing only system info with a read-only community. A common “oops” is exposing the full tree broadly.

---

## Dangerous Settings

These are the types of settings that can turn SNMP into an information-leak (or worse):

* `rwuser noauth`
  Gives write access without proper authentication controls.

* `rwcommunity <string> <IPv4>` / `rwcommunity6 <string> <IPv6>`
  Grants write access based on community string, often too broadly (or even to all sources).

Anything that enables **write** access widely is a serious risk, because it may allow configuration changes and service disruption.

---

# Footprinting SNMP

## 1) Discover / Confirm SNMP is reachable

A basic port scan is usually enough:

* UDP/161 open suggests SNMP is available
* UDP/162 suggests traps may be configured (less commonly exposed)

## 2) Enumerate with `snmpwalk` (once you have a community string)

Example (v2c, community `public`):

```bash
snmpwalk -v2c -c public 10.129.14.128
```

Typical high-value leaks you may see:

* OS name and kernel version
* Hostname
* Contact email / admin identifier
* Running processes / installed packages (host-resources MIB)
* Interface information and network details

## 3) Brute-force community strings (if unknown)

`onesixtyone` is commonly used with wordlists (for example from SecLists):

```bash
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128
```

This is often effective because many environments still rely on predictable community strings.

## 4) Fast OID sweeping with `braa`

Once you have a community string, `braa` can rapidly query OIDs:

```bash
braa public@10.129.14.128:.1.3.6.*
```

This is useful to quickly identify “interesting” branches without doing a full walk.

---

## Practical Takeaways

* SNMP is often one of the **fastest ways to learn what a host is** (OS, hostname, services hints).
* **v1/v2c are high-risk** if reachable from untrusted networks due to plaintext community strings.
* Misconfigurations can expose internal details that feed directly into:

  * Service/version targeting
  * User enumeration (contacts/emails)
  * Package/process discovery (attack surface mapping)

If you want, paste the next section you are working on (or tell me the module style you want), and I will rewrite it in the same clean, structured format as above.
