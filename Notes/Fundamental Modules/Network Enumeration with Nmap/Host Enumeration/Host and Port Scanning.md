## Host and Port Scanning

Once you have confirmed that a target is alive, the next step is to build a more complete profile of the system. This typically includes:

* Open ports and associated services
* Service versions and banners
* Information returned by services (configuration, features, metadata)
* Operating system indicators (fingerprinting)

To interpret scan results correctly, you need to understand **how Nmap determines port states** and what each state means.

---

## Port States in Nmap

Nmap can report six possible states for a scanned port:

| State              | Description                                                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| `open`             | The target accepted the probe (TCP connection, UDP response, or SCTP association). An application is likely listening.           |
| `closed`           | The target responded but no service is listening (typically TCP RST). This also proves the host is reachable.                    |
| `filtered`         | Nmap cannot determine if the port is open or closed because packets are being dropped or blocked (no response or ICMP error).    |
| `unfiltered`       | Seen mainly with TCP ACK scans; the port is reachable, but Nmap cannot determine open vs closed.                                 |
| `open\|filtered`   | No response was received; the port may be open or filtered by a firewall/packet filter. Common with UDP and some TCP scan types. |
| `closed\|filtered` | Seen mainly with IP ID idle scans; Nmap cannot determine if the port is closed or filtered.                                      |

---

## Discovering Open TCP Ports

By default:

* Nmap scans the **top 1000 TCP ports**
* If run as root, default TCP scan is **SYN scan** (`-sS`)
* If not root, default is **Connect scan** (`-sT`) because raw packet creation requires elevated privileges

### Common ways to select ports

* Specific ports: `-p 22,25,80,139,445`
* Port range: `-p 22-445`
* Top ports: `--top-ports=10`
* All ports: `-p-`
* Fast scan (top 100 ports): `-F`

---

## Scanning Top 10 TCP Ports

```
sudo nmap 10.129.2.28 --top-ports=10
```

This scans only the most frequently seen TCP ports in Nmap’s internal database and reports their state.

| Option           | Meaning                           |
| ---------------- | --------------------------------- |
| `10.129.2.28`    | Target host                       |
| `--top-ports=10` | Scan the top 10 most common ports |

---

## Packet Tracing a SYN Scan

If you want to observe what is happening at packet level, you can trace sent and received packets. Commonly you also disable extra discovery methods so the output is easier to interpret:

```
sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping
```

| Option               | Meaning                                |
| -------------------- | -------------------------------------- |
| `-p 21`              | Scan only port 21                      |
| `--packet-trace`     | Show packets sent/received             |
| `-Pn`                | Skip host discovery (treat host as up) |
| `-n`                 | Disable DNS resolution                 |
| `--disable-arp-ping` | Do not use ARP to discover host        |

### Interpreting the trace

* `S` in the SENT line means **SYN**
* `RA` in the RCVD line means **RST + ACK**

  * RST indicates “nothing is listening” (port closed)
  * ACK acknowledges receipt of the SYN

A typical interpretation:

* SYN sent → RST/ACK received → **port is closed**
* SYN sent → SYN/ACK received → **port is open**
* SYN sent → no response → **filtered** or **open|filtered** (depends on scan type)

---

## Connect Scan (`-sT`)

A TCP Connect scan completes the full three-way handshake. It is accurate and works without raw sockets, but it is noisier and generally easier to log and detect.

Example:

```
sudo nmap 10.129.2.28 -p 443 --packet-trace --disable-arp-ping -Pn -n --reason -sT
```

| Option     | Meaning                              |
| ---------- | ------------------------------------ |
| `-sT`      | Full TCP connect scan                |
| `--reason` | Show the reason Nmap chose the state |

If the output shows `syn-ack` as the reason, it confirms the port responded with SYN/ACK, indicating it is reachable and likely listening.

---

## Filtered Ports

A port often appears `filtered` when a firewall or ACL is interfering. Two common firewall behaviours:

### 1) Dropping packets (silent drop)

Nmap receives no reply, waits, and retries.

Example (port 139):

```
sudo nmap 10.129.2.28 -p 139 --packet-trace -n --disable-arp-ping -Pn
```

You typically see repeated SYN packets and no response. Scan takes longer because Nmap is waiting and retrying.

### 2) Rejecting packets (explicit response)

Instead of silence, the firewall or host responds with an ICMP error (or sometimes TCP RST depending on rule/device).

Example (port 445 showing ICMP unreachable):

```
sudo nmap 10.129.2.28 -p 445 --packet-trace -n --disable-arp-ping -Pn
```

If you receive ICMP type 3 code 3 (port unreachable) in response to your probe, the traffic is being actively rejected (or the host is signalling it cannot be reached on that port/protocol).

Practical takeaway: if you know the host is alive but a port is `filtered`, that port is still worth investigating later using alternate techniques (different probes, different timing, different scan type, or service-specific enumeration from an allowed network position).

---

## Discovering Open UDP Ports

UDP scanning is slower and more ambiguous because UDP is stateless. Many UDP services **do not respond** to empty datagrams, even when the port is open.

Fast UDP scan example:

```
sudo nmap 10.129.2.28 -F -sU
```

| Option | Meaning                   |
| ------ | ------------------------- |
| `-sU`  | UDP scan                  |
| `-F`   | Fast mode (top 100 ports) |

### Why UDP often shows `open|filtered`

* If a UDP service responds, Nmap can mark it `open`
* If a device sends ICMP type 3 code 3 (port unreachable), Nmap marks it `closed`
* If there is no response at all, Nmap often marks it `open|filtered`

Packet trace example showing a real UDP response:

```
sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 137 --reason
```

If you get a UDP response back, Nmap can confidently label the port `open`.

Closed UDP example (ICMP port unreachable):

```
sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 100 --reason
```

Open|filtered example (no response):

```
sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 138 --reason
```

---

## Service Version Detection (`-sV`)

Once ports are identified, you typically want service information (service name, version, and sometimes extra metadata). Nmap does this by sending probes and matching responses.

Example:

```
sudo nmap 10.129.2.28 -Pn -n --disable-arp-ping --packet-trace -p 445 --reason -sV
```

| Option           | Meaning                                       |
| ---------------- | --------------------------------------------- |
| `-sV`            | Service/version detection                     |
| `--packet-trace` | Useful to observe probes and timeouts         |
| `--reason`       | Shows what response caused the classification |

Important technical note: version detection can be slow on some services because Nmap sends multiple probes and waits for responses. This is why you may see reads timing out and then Nmap trying a different probe until it matches a known service fingerprint.

---

## Additional Technical Notes

### Default retry behaviour

Nmap retries when it gets no response, especially when scanning through packet loss or filtering. The default retry behaviour is controlled by `--max-retries`. Reducing retries can speed scans but may increase false negatives.

### Timing templates

Nmap timing templates (`-T0` to `-T5`) change scan aggressiveness. Faster scans can be noisier and less reliable on unstable networks. Slower scans are often more accurate and stealthier.

### Practical scanning flow

A common workflow in internal assessments is:

1. Host discovery (`-sn`) to build a live host list
2. TCP quick scan (`--top-ports` or `-F`) to prioritise targets
3. Full TCP scan on selected hosts (`-p-`)
4. Service detection (`-sV`) on identified ports
5. Scripted/service-specific enumeration (NSE scripts, manual probes)

---

More information on Nmap port scanning techniques:

[https://nmap.org/book/man-port-scanning-techniques.html](https://nmap.org/book/man-port-scanning-techniques.html)
