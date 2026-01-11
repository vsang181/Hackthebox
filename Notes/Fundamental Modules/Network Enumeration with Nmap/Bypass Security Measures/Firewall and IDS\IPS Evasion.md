## Firewall and IDS/IPS Evasion

Nmap includes multiple techniques that can help you **bypass firewall rules** and reduce the chance of being detected by **IDS/IPS**. These techniques include:

* Packet fragmentation
* Decoys and spoofing
* Alternative scan types (for example, ACK scans)
* Source port manipulation
* DNS proxying and custom resolvers
* Timing and rate control (covered previously)

The effectiveness of each approach depends on **where filtering happens** (host firewall, perimeter firewall, router ACLs, WAF, IPS inline, etc.) and **what the security controls actually inspect** (headers only vs deep packet inspection).

---

## Firewalls

A firewall is a security control designed to prevent unauthorised traffic by enforcing rules that decide whether packets are:

* Allowed
* Dropped (silently ignored)
* Rejected (explicitly denied)

Firewalls commonly filter based on:

* Source/destination IP
* Source/destination port
* Protocol (TCP/UDP/ICMP)
* Connection state (stateful vs stateless)
* Application signatures (NGFW)

---

## IDS and IPS

IDS and IPS are monitoring and enforcement controls:

* **IDS (Intrusion Detection System)** observes traffic and alerts on suspicious behaviour.
* **IPS (Intrusion Prevention System)** sits inline (or otherwise enforces) and can block or disrupt traffic automatically.

They often use:

* Signature/pattern matching
* Behaviour-based rules (rate, scan patterns)
* Protocol validation (malformed packets, weird flags)
* Reputation lists and geo rules

---

## Determining Firewall Behaviour (Dropped vs Rejected)

When a port is reported as **filtered**, it generally means Nmap cannot determine whether it is open or closed because traffic is being filtered.

Two common firewall behaviours:

### Dropped packets

* No response is returned.
* Nmap retries probes multiple times and eventually labels the port as filtered or open|filtered.

### Rejected packets

* The host or firewall replies explicitly.

* TCP may return **RST**

* ICMP may return errors such as:

* Net Unreachable

* Net Prohibited

* Host Unreachable

* Host Prohibited

* Port Unreachable

* Proto Unreachable

This difference matters because it gives you insight into **how strict the filtering is** and whether you can infer reachability.

---

## ACK Scan (-sA) to Map Filtering

The **TCP ACK scan (`-sA`)** is commonly used to help determine whether a firewall is filtering traffic.

* It sends TCP packets with only the **ACK** flag set.
* Many firewalls are stricter with **new inbound SYN** connections than with packets that look like they belong to an existing session.

Expected behaviour:

* If the packet reaches the host (open or closed port), the host typically replies with **RST**.
* If no reply is received, Nmap assumes the packet was blocked/filtered.

### Comparing SYN scan vs ACK scan

SYN scan (`-sS`) typically shows port states such as:

* `open` (SYN-ACK received)
* `closed` (RST received)
* `filtered` (no response or ICMP filtering)

ACK scan (`-sA`) typically results in:

* `unfiltered` (RST received, meaning firewall let it through)
* `filtered` (no response, likely blocked)

This does not tell you whether the port is open. It tells you whether the firewall is filtering.

---

## Detecting IDS/IPS Presence

IDS/IPS detection is harder than firewall detection because IDS can be passive and IPS may block selectively.

Indicators you may be dealing with IPS-like behaviour:

* You get responses initially, then later scans stop working
* Only specific scan types are blocked (for example, `-sV`, `-A`, or NSE scripts)
* You are rate-limited or temporarily blacklisted after bursts of probes
* Responses become inconsistent (timeouts increase, resets appear)

Operational reality during assessments:

* If your source IP gets blocked, you usually have to switch origin (for example, another testing host/VPS) or slow down significantly.

---

## Decoys (-D)

Decoy scanning helps obscure the real scanning source by mixing your traffic with spoofed sources.

```
-D RND:5
```

* Nmap generates random IP addresses and places your real IP among them.
* The goal is to make logs less clear about who initiated the scan.

Important limitations:

* Many networks and ISPs drop spoofed packets (anti-spoofing / uRPF).
* If decoy sources are “dead” or unreachable, some targets may behave unexpectedly (and defensive systems may treat it as suspicious volume).

Decoys can be used with multiple scan types such as:

* SYN scans
* ACK scans
* ICMP-based discovery
* OS detection scans

---

## Source IP Spoofing (-S) and Interface Selection (-e)

You can test whether access differs by subnet/source by spoofing a source IP:

```
-S <source_ip> -e <interface>
```

This is typically only useful when:

* You are inside the same routed environment where spoofing is not blocked, or
* You control the routing environment sufficiently to make replies come back to you

In many real networks, spoofing fails because replies go to the spoofed address, not you.

---

## Source Port Manipulation (--source-port)

Some firewalls trust traffic that *appears* to originate from common allowed ports (for example DNS `53`, HTTP `80`, or HTTPS `443`).

You can force Nmap to originate traffic from a chosen source port:

```
--source-port 53
```

This sometimes works when firewall rules are written too broadly, such as:

* “Allow outbound/inbound DNS” without strict state tracking
* Legacy ACLs that trust “service ports” rather than flow legitimacy

Your example demonstrates:

* Port appears filtered normally
* Becomes reachable when using `--source-port 53`

This is an example of a **rule weakness**, not a guaranteed bypass.

---

## DNS Proxying and Custom DNS Servers (--dns-server)

By default, Nmap may perform reverse DNS lookups unless disabled (`-n` disables DNS).

You can specify a resolver:

```
--dns-server <ns1>,<ns2>
```

Why this can matter:

* In segmented environments (for example, DMZ), internal resolvers may be trusted or allowed where external resolvers are not.
* DNS traffic may traverse paths that other traffic cannot.

Also note:

* Modern DNS increasingly uses TCP/53 more often (DNSSEC, larger responses, IPv6, etc.), which may influence firewall behaviour.

---

## Additional Technical Notes

### Fragmentation (-f / --mtu)

Nmap can fragment probes so packet inspection devices may not reassemble properly:

* `-f` fragments packets into small IP fragments
* `--mtu <n>` sets a custom MTU (must be a multiple of 8 for IP fragments)

This can sometimes evade simplistic filtering, but modern IDS/IPS commonly reassemble.

### Bad checksum (--badsum)

Sends packets with incorrect checksums:

* Some IDS might log them
* Targets usually drop them
* Useful to test whether a device is passively monitoring versus the host actually receiving traffic

### Idle scan (-sI)

A stealth technique that uses a “zombie” host to scan indirectly. Very situational, but useful when it works.

---

## Practical Strategy When Evasion Is Needed

A common workflow:

1. Identify whether filtering is “drop” or “reject”
2. Use `-sA` to map what is filtered vs reachable
3. Try low-noise host discovery alternatives (`-PS`, `-PA`, `-PU`, `-PE`, `-Pn` as needed)
4. Test source-port manipulation on a small set of ports
5. Slow down (`-T2/-T3`, reduce retries, reduce parallelism) if IPS behaviour is suspected
6. Only then consider fragmentation/decoys if appropriate
