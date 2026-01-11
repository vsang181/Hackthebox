## Host Discovery

Before you start enumerating services or scanning ports, you need a clear picture of **which systems are actually online**. In an internal assessment, this usually means identifying all reachable hosts inside a given network range. Nmap provides several host discovery techniques to help you do exactly that.

Your goal here is simple: build a reliable list of live systems that are worth further investigation.

It is good practice to **save every scan you run**. Stored scan results are useful for documentation, reporting, and comparison. Different tools and techniques can produce different outcomes, so keeping raw output helps you understand why results differ and which method revealed what.

---

## Scanning a Network Range

One of the most common tasks is discovering live hosts in a subnet.

```
sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5
```

Example output:

```
10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

### Key options used

* `10.129.2.0/24`
  Targets the entire subnet.

* `-sn`
  Disables port scanning and performs host discovery only.

* `-oA tnet`
  Saves output in all formats (`.nmap`, `.gnmap`, `.xml`) using the prefix `tnet`.

This approach works well **only if ICMP or ARP traffic is not blocked**. If a firewall drops these packets, hosts may appear offline even when they are not.

---

## Scanning from an IP List

In many engagements, you will be given a predefined list of IP addresses rather than a full subnet. Nmap can read targets directly from a file.

Example list:

```
10.129.2.4
10.129.2.10
10.129.2.11
10.129.2.18
10.129.2.19
10.129.2.20
10.129.2.28
```

Scan the list like this:

```
sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5
```

Example result:

```
10.129.2.18
10.129.2.19
10.129.2.20
```

Only three hosts respond. This does **not automatically mean the others are offline**. It may simply indicate that those systems ignore ICMP echo requests or ARP probes due to firewall rules.

---

## Scanning Multiple Individual IPs

If you only need to check a small set of systems, you can specify them directly:

```
sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20 | grep for | cut -d" " -f5
```

If the addresses are sequential, you can shorten this:

```
sudo nmap -sn -oA tnet 10.129.2.18-20 | grep for | cut -d" " -f5
```

This is useful when validating a shortlist of targets before deeper scans.

---

## Scanning a Single Host

Before running port or service scans against a host, you should confirm that it is alive.

```
sudo nmap 10.129.2.18 -sn -oA host
```

Example output:

```
Nmap scan report for 10.129.2.18
Host is up (0.087s latency).
MAC Address: DE:AD:00:00:BE:EF
```

By default, when port scanning is disabled, Nmap attempts **ARP discovery first** on local networks. This is why earlier scans succeeded even without ICMP replies.

---

## Forcing ICMP Echo Requests

To explicitly observe ICMP behaviour, you can enable packet tracing and ICMP echo requests:

```
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace
```

This shows every packet sent and received, allowing you to confirm how Nmap determines host availability.

To see *why* Nmap considers a host alive, use:

```
sudo nmap 10.129.2.18 -sn -oA host -PE --reason
```

Example result:

```
Host is up, received arp-response
```

This confirms the host was detected purely through ARP, not ICMP.

---

## Disabling ARP and Using ICMP Only

If you want to avoid ARP entirely and rely solely on ICMP echo requests, disable ARP pings:

```
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping
```

Now you can clearly see ICMP echo requests and replies being exchanged. This is especially useful when testing firewall behaviour or troubleshooting false negatives during host discovery.

