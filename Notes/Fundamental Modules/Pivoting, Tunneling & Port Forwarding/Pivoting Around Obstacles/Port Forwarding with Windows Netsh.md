# Port Forwarding with Windows Netsh

[Netsh](https://docs.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-contexts) is a built-in Windows command-line tool that among other things lets you create persistent port forwarding rules through its `interface portproxy` subcommand.  Because it is a native Windows binary, using it for pivoting is far less likely to trigger endpoint protection than uploading a third-party tunneling binary -- making it a strong living-off-the-land option. 

## How Portproxy Works

The `v4tov4` portproxy rule tells Windows to listen on a specific IP and port, and when a connection arrives, open a separate TCP connection to a configured destination IP and port, then relay traffic between the two.  All forwarded connections will appear on the destination as originating from the Windows pivot host itself, not the original attacker IP, since netsh portproxy does not preserve the source address. 

Administrator-level privileges are required to create portproxy rules. The rules are written to the Windows registry under `HKLM:\SYSTEM\CurrentControlSet\Services\PortProxy\v4tov4\tcp` and survive reboots by default. 

## Setting Up the Port Forward

In this scenario, the Windows workstation sits at `10.129.15.150` (reachable from the attack host) and also has access to `172.16.5.0/23` internally. The goal is to reach RDP on `172.16.5.25:3389` through it:

```cmd
C:\Windows\system32> netsh.exe interface portproxy add v4tov4 listenport=8080 listenaddress=10.129.15.150 connectport=3389 connectaddress=172.16.5.25
```

Breaking down the arguments:

- `listenaddress=10.129.15.150` -- the IP on the Windows pivot host to bind the listener to (its external-facing IP)
- `listenport=8080` -- the port the attack host will connect to on the pivot
- `connectaddress=172.16.5.25` -- the internal target to forward traffic to
- `connectport=3389` -- the RDP port on the internal target

Verify the rule was created correctly:

```cmd
C:\Windows\system32> netsh.exe interface portproxy show v4tov4

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
10.129.15.150   8080        172.16.5.25     3389
```

## Allowing Traffic Through the Windows Firewall

Creating a portproxy rule alone is not enough -- the Windows firewall will block inbound connections to port 8080 unless an explicit rule is added.  Without this step, the attacker's connection attempt will be silently dropped at the pivot host: 

```cmd
netsh advfirewall firewall add rule name="pivot-8080" protocol=TCP dir=in localport=8080 action=allow
```

This can also be done via PowerShell on modern Windows versions:

```powershell
New-NetFirewallRule -Name "pivot-8080" -DisplayName "pivot-8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow -Enabled True
```

## Connecting from the Attack Host

With the portproxy and firewall rule in place, xfreerdp connects to the Windows pivot host's external IP on port 8080, and netsh silently relays the RDP session to `172.16.5.25:3389`:

```bash
xfreerdp /v:10.129.15.150:8080 /u:victor /p:password
```

The RDP session connects to the internal host, not the pivot -- the Windows workstation is entirely transparent once the rule is active.

## Cleanup

Since portproxy rules persist after reboot, removing them when the engagement is complete is important to avoid leaving artifacts: 

```cmd
# Remove a specific rule
netsh interface portproxy delete v4tov4 listenport=8080 listenaddress=10.129.15.150

# Remove all portproxy rules at once
netsh interface portproxy reset

# Remove the firewall rule
netsh advfirewall firewall del rule name="pivot-8080"
```
