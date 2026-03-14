# Attacking RDP

[Remote Desktop Protocol (RDP)](https://en.wikipedia.org/wiki/Remote_Desktop_Protocol) runs on TCP/3389 by default and is one of the most widely used remote administration tools in enterprise environments. Its prevalence also makes it a consistent target -- every exposed RDP service is a potential foothold, a lateral movement path, or a session worth hijacking depending on the stage of an engagement.

## Enumeration

A basic Nmap scan confirms whether RDP is listening:

```bash
nmap -Pn -p3389 192.168.2.143
```

The `-Pn` flag skips host discovery and treats the target as up, which is useful when ICMP is blocked. A state of `open` on port 3389 confirms the service is active and reachable.

## Misconfigurations and Password Attacks

RDP authenticates with user credentials, making it a direct target for password spraying. Full brute-forcing is generally inadvisable because most Windows environments lock accounts after a defined number of failed attempts. Password spraying -- one or two passwords tested across many usernames with deliberate pauses between rounds -- avoids triggering lockouts.

[Crowbar](https://github.com/galkan/crowbar) is purpose-built for RDP spraying:

```bash
crowbar -b rdp -s 192.168.220.142/32 -U users.txt -c 'password123'
```

Hydra also supports RDP, though it should be run with reduced parallelism since RDP servers are sensitive to many simultaneous connections:

```bash
# Use -t 1 or -t 4 to reduce parallel threads
hydra -L usernames.txt -p 'password123' 192.168.2.143 rdp
```

Once valid credentials are identified, connecting is straightforward using either `rdesktop` or `xfreerdp`:

```bash
rdesktop -u admin -p password123 192.168.2.143
```

Accept any certificate warning by typing `yes` when prompted.

## RDP Session Hijacking

Once local administrator access is established on a machine where other users are connected via RDP, those sessions can be hijacked without needing the target user's credentials. Windows' built-in [tscon.exe](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/tscon) connects one session to another -- and when executed as SYSTEM, Windows performs no authentication check before completing the transfer.

The first step is identifying active sessions:

```cmd
query user
```

This returns each logged-on user, their session name, and their session ID. The goal is to connect the target user's session ID to your own session name.

The direct approach requires SYSTEM privileges. One reliable method when you hold local administrator rights is creating a Windows service, which runs as Local System (SYSTEM) by default:

```cmd
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

Breaking this down:
- `tscon 2` specifies the target session ID to take over
- `/dest:rdp-tcp#13` specifies your current session name as the destination

When the service starts, a new terminal appears running under the hijacked user's session. No password prompt is presented because SYSTEM is trusted to manage all sessions on the host.

This technique is catalogued under [MITRE ATT&CK T1563.002](https://attack.mitre.org/techniques/T1563/002/). It is worth noting that this method no longer functions on Windows Server 2019.

## RDP Pass-the-Hash

When only an NTLM hash is available -- for example from a SAM database dump -- and the plaintext cannot be cracked, RDP can still be accessed using Pass-the-Hash via `xfreerdp` with the `/pth` flag:

```bash
xfreerdp /v:192.168.220.152 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

There is one prerequisite: the target host must have Restricted Admin Mode enabled. This mode prevents credentials from being transmitted over the network during RDP authentication, which is what makes hash-based login possible. It is disabled by default and must be enabled by adding a registry key -- either from an existing session on the target or remotely if registry access is available:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Setting `DisableRestrictedAdmin` to `0x0` enables the feature. Once set, the `xfreerdp /pth` command will complete the connection using the hash in place of a password.

Two caveats apply to this technique:

- It only works for accounts in the local Administrators group. Accounts that are members of the Remote Desktop Users group but not local admins cannot use Restricted Admin Mode
- Not every Windows version or configuration will permit this, so it is worth attempting when the conditions are met rather than assuming it will always succeed

## Attack Chain Summary

RDP attacks tend to chain together naturally during an engagement. A password spray produces initial access. Local administrator rights on that host enable a SAM dump. SAM hashes can be used for Pass-the-Hash against RDP on other machines in the network. If active sessions exist on any compromised host, session hijacking can elevate access to whatever account those users hold -- potentially a Domain Admin or a service account with broader network rights than the originally compromised user.
