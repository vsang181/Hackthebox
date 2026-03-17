# SSH Pivoting with Sshuttle

[Sshuttle](https://github.com/sshuttle/sshuttle) takes a fundamentally different approach to SSH-based pivoting compared to everything covered so far. Rather than creating a SOCKS proxy that requires proxychains to intercept tool traffic, sshuttle manipulates iptables directly on the attacker's machine to transparently route all traffic destined for a target subnet through the SSH tunnel. The result behaves like a VPN -- tools run without any prefix or proxy configuration, and network traffic to the specified subnets is automatically redirected.

## How It Works Under the Hood

When sshuttle runs, it connects to the remote host over SSH and starts a small Python server on the pivot host. On the attacker's machine, it creates iptables NAT rules that intercept all outbound TCP packets destined for the specified subnets and redirect them to a local TCP redirector, which forwards them through the SSH session. The verbose output confirms this:

```
fw: iptables -w -t nat -N sshuttle-12300
fw: iptables -w -t nat -I OUTPUT 1 -j sshuttle-12300
fw: iptables -w -t nat -A sshuttle-12300 -j REDIRECT --dest 172.16.5.0/32 -p tcp --to-ports 12300
```

This is why no proxychains configuration is needed -- the redirection happens at the kernel networking level before any application sees the packet.

## Requirements

Sshuttle has three hard requirements that must be met on the pivot host:

- SSH access with a valid account
- Python 3 installed on the pivot host (sshuttle runs a server component there)
- The attacker's machine must be Linux-based (sshuttle manipulates iptables, which is Linux-specific)

It does not work from a Windows attack host, and it only supports SSH as the transport. For TOR or HTTP/HTTPS proxy-based tunneling, proxychains with dynamic port forwarding remains the appropriate tool.

## Installation and Usage

```bash
sudo apt-get install sshuttle
```

Once installed, a single command sets up the entire tunnel and iptables routing:

```bash
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
```

Breaking down the flags:

- `-r ubuntu@10.129.202.64` -- the remote pivot host to SSH into
- `172.16.5.0/23` -- the subnet to route through the pivot
- `-v` -- verbose mode, showing iptables rules and connection activity as they happen

After entering the SSH password, sshuttle confirms the tunnel is active and the firewall rules are in place. From this point, any tool on the attack host can target `172.16.5.0/23` addresses directly with no additional configuration:

```bash
# No proxychains needed
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn
```

The Nmap output returns full service detail including RDP certificate information, hostname (`DC01.inlanefreight.local`), domain FQDN, and OS version -- the same result that would require proxychains with the dynamic port forwarding approach, but with cleaner syntax and no tool-specific workarounds.

## Sshuttle vs Proxychains with Dynamic Port Forwarding

| Aspect | Sshuttle | SSH -D with Proxychains |
|---|---|---|
| Tool prefix required | No -- tools run natively | Yes -- `proxychains` before every command |
| Scan types supported | Full TCP connect and others | Full TCP connect only |
| Transport options | SSH only | SSH, TOR, HTTP/HTTPS proxies |
| Attack host requirement | Linux only | Linux or Windows (with plink) |
| Pivot host requirement | Python 3 + SSH | SSH only |
| Setup complexity | Single command | Two steps (SSH tunnel + proxychains config) |

Sshuttle is the faster and more convenient option when the pivot host has Python installed and the attack host is Linux. The proxychains approach is more flexible when working with non-SSH tunnels, Windows attack hosts, or environments where installing packages on the pivot is not possible.
