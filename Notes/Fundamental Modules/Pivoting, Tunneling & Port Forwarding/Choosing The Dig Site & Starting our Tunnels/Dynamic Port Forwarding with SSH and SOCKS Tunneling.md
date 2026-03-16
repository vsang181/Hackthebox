# Dynamic Port Forwarding with SSH and SOCKS Tunneling

Port forwarding and SOCKS tunneling are among the most practical tools in a penetration tester's post-exploitation toolkit. This section covers two distinct techniques -- SSH local port forwarding for accessing specific services, and SSH dynamic port forwarding for routing entire tool suites through a compromised host into an otherwise unreachable network.

## SSH Local Port Forwarding

Local port forwarding binds a port on your attack host and forwards any traffic sent to it through the SSH connection to a destination the pivot host can reach. The syntax is:

```bash
ssh -L <local_port>:<destination_host>:<destination_port> <user>@<pivot_host>
```

In practice, after scanning the target Ubuntu server and confirming MySQL (3306) is closed to the outside but SSH (22) is open, local port forwarding exposes MySQL on the attacker's own localhost:

```bash
ssh -L 1234:localhost:3306 ubuntu@10.129.202.64
```

This tells the SSH client to forward all data arriving on the attacker's local port 1234 to `localhost:3306` from the Ubuntu server's perspective. Because `localhost` here means the Ubuntu server itself, the MySQL instance that was unreachable externally is now accessible on the attacker's machine at `127.0.0.1:1234`.

Verification can be done two ways:

```bash
# Confirm the listener is active
netstat -antp | grep 1234

# Confirm the service behind the forward
nmap -v -sV -p1234 localhost
```

The Nmap output against localhost:1234 will identify the service as MySQL, confirming the forward is working correctly.

Multiple ports can be forwarded in a single SSH command by chaining `-L` flags:

```bash
ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@10.129.202.64
```

This is useful when multiple services need to be reached simultaneously during a single session.

## Identifying the Pivot Opportunity

Running `ifconfig` on the compromised Ubuntu server reveals the dual-homed configuration that makes dynamic pivoting possible:

```
ens192: 10.129.202.64   (reachable from the attacker)
ens224: 172.16.5.129    (connected to an internal network the attacker cannot reach)
lo:     127.0.0.1
```

The `ens224` interface with address `172.16.5.129` in the `172.16.5.0/23` range is the indicator. The attacker's machine has no route to that subnet, but the Ubuntu server does -- making it an ideal pivot point for dynamic forwarding.

## SSH Dynamic Port Forwarding and SOCKS Tunneling

Unlike local port forwarding which maps one port to one destination, dynamic port forwarding creates a SOCKS proxy that can route traffic to any destination reachable by the pivot host. A single command sets this up:

```bash
ssh -D 9050 ubuntu@10.129.202.64
```

The `-D 9050` flag instructs the SSH server to allow dynamic port forwarding, and the SSH client begins listening on localhost:9050 as a SOCKS proxy. Any traffic sent to that port is forwarded through the SSH tunnel and can reach the entire `172.16.5.0/23` network via the Ubuntu server.

[SOCKS](https://en.wikipedia.org/wiki/SOCKS) (Socket Secure) is a protocol that proxies TCP connections on behalf of a client, making it compatible with any TCP-capable tool. SOCKS4 provides basic functionality without authentication or UDP support. SOCKS5 adds authentication and UDP support.

## Configuring Proxychains

Proxychains intercepts outbound TCP connections from any tool and routes them through the configured SOCKS proxy. Confirm the configuration points to the correct port:

```bash
tail -4 /etc/proxychains.conf
# Should contain:
socks4  127.0.0.1 9050
```

Once configured, prefix any tool with `proxychains` to route its traffic through the tunnel:

```bash
# Host discovery across the internal subnet
proxychains nmap -v -sn 172.16.5.1-200

# Full port scan against a specific internal host
proxychains nmap -v -Pn -sT 172.16.5.19
```

Two important constraints apply when scanning through proxychains:

- Only full TCP connect scans work (`-sT`). Proxychains cannot handle partial packets, so half-open SYN scans (`-sS`) will return incorrect results
- ICMP ping checks do not work against Windows targets because Windows Defender blocks ICMP by default, so always use `-Pn` to skip host discovery

## Using Metasploit Through the Tunnel

Metasploit can be launched directly through proxychains, routing all module traffic through the SOCKS proxy:

```bash
proxychains msfconsole
```

With Metasploit running through the tunnel, auxiliary modules can target internal hosts directly. For example, scanning for RDP on the internal Windows host:

```bash
msf6 > use auxiliary/scanner/rdp/rdp_scanner
msf6 auxiliary(scanner/rdp/rdp_scanner) > set rhosts 172.16.5.19
msf6 auxiliary(scanner/rdp/rdp_scanner) > run
```

The proxychains output in the terminal confirms each connection attempt routes through `127.0.0.1:9050` before reaching the internal target. The scanner output confirms RDP is running on `172.16.5.19` with the hostname `DC01`, domain `DC01`, and Windows version `10.0.17763`, with NLA not required.

## Connecting via RDP Through the Tunnel

With valid credentials in hand, xfreerdp can be routed through proxychains to establish a full GUI session on the internal Windows host:

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

After accepting the RDP certificate, a full desktop session opens on the internal Windows machine, reached entirely through the SOCKS tunnel via the Ubuntu pivot host -- without any direct network route from the attacker's machine to `172.16.5.0/23`.
