# The Networking Behind Pivoting

Before any pivoting technique can be applied effectively, the underlying networking concepts need to be solid. The decisions made during pivoting -- which host to use, which route to add, which port to forward traffic through -- all depend on reading and interpreting the network information available on a compromised host. 

## IP Addressing and NICs

Every device communicating on a network requires at least one IP address, assigned to a Network Interface Controller (NIC). A host can have multiple NICs -- both physical and virtual -- each assigned to a different network. This is precisely what makes dual-homed hosts so valuable as pivot candidates: they have native connectivity to two or more network segments simultaneously. 

The commands to enumerate NICs and their addresses:

```bash
# Linux/macOS
ifconfig

# Windows
ipconfig
```

Reading `ifconfig` output from a Linux pivot host reveals several important details per interface:

| Interface | What It Tells You |
|---|---|
| `eth0` with a public IP (e.g. 134.122.100.200) | Host is directly internet-facing, likely in a DMZ |
| `eth1` with a private IP (e.g. 10.106.0.172) | Host has connectivity to an internal network segment |
| `lo` at 127.0.0.1 | Loopback only -- no network significance for pivoting |
| `tun0` with a 10.x.x.x address | Active VPN tunnel -- traffic to that range routes through it |

The presence of `eth1` alongside `eth0` is the clearest indicator of a pivot opportunity -- it means the host can reach a network segment that the attacker's machine cannot access directly.

On Windows, `ipconfig` shows similar information. Dual-stack configurations are common, where a NIC holds both an IPv4 and an IPv6 address. This module focuses primarily on IPv4, which remains the dominant addressing scheme in enterprise LANs, but IPv6 addresses are worth noting as they may expose additional attack paths.

## Routing

Every host -- not just dedicated routers -- maintains a routing table that governs where outbound packets are sent based on their destination IP. When a packet is generated, the OS consults this table to determine which NIC and gateway to use. 

```bash
# Linux -- either command works
netstat -r
ip route
```

Reading a routing table entry:

```
Destination     Gateway         Genmask         Flags   Iface
10.10.10.0      10.10.14.1      255.255.254.0   UG      tun0
10.106.0.0      0.0.0.0         255.255.240.0   U       eth1
default         178.62.64.1     0.0.0.0         UG      eth0
```

Breaking this down:
- Traffic to `10.10.10.0/23` is forwarded through gateway `10.10.14.1` via the `tun0` tunnel interface
- Traffic to `10.106.0.0/20` is sent directly out of `eth1` (no gateway needed, same subnet)
- Everything else goes to `178.62.64.1` via `eth0` as the default gateway

For pivoting, two things matter in the routing table. First, it reveals which networks the pivot host can already reach -- any destination listed is accessible. Second, adding a route through a pivot host is how tools like Metasploit's AutoRoute extend the attacker's reach into those segments without needing a full VPN.  When a route for a target subnet is added through the pivot host's IP, the attacker's machine begins forwarding packets for that destination through the pivot automatically.

Any destination not covered by a specific route falls to the default gateway. This is also worth knowing -- if the segment you want to reach has no route on the pivot host, traffic to it will leave via the default gateway and likely never arrive.

## Protocols, Services, and Ports

Ports act as identifiers for specific applications running on a host. This matters for pivoting because firewalls typically filter based on ports and protocols rather than the content of traffic. A firewall that blocks port 4444 (a common Meterpreter default) will not block port 443, which is why tunneling traffic through HTTPS or SSH is so effective -- the firewall sees traffic it is already expected to allow.

Three considerations when planning pivoting through ports:

- **Inbound firewall rules on the pivot host** -- determines which ports on the pivot are reachable from the attacker's machine
- **Outbound firewall rules from the pivot** -- determines which ports the pivot can connect to on internal targets
- **Source port tracking** -- the OS assigns a source port for each established connection, which needs to be accounted for when setting up listeners and ensuring callbacks reach the correct service

A web server listening on port 80 is an example of a naturally permitted channel. Administrators cannot block inbound port 80 without taking down the website -- which means that port can often be used to deliver payloads or maintain C2, blending with the legitimate traffic already passing through it. 

## Practical Documentation Tip

As networks become more complex during an engagement, keeping track of which hosts exist, which interfaces they have, and which segments they can reach becomes essential. Drawing the network topology -- even a rough diagram in a tool like [Draw.io](https://draw.io/) -- prevents confusion when planning multi-hop pivots and also serves as documentation for the final report. For each host compromised, recording the NIC addresses and routing table entries provides the raw material needed to map the full picture of what is reachable and through which path. 
