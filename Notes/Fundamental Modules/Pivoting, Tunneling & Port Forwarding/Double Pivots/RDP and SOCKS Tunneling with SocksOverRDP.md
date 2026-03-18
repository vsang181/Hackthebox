# RDP and SOCKS Tunneling with SocksOverRDP

[SocksOverRDP](https://github.com/nccgroup/SocksOverRDP) is a tool developed by NCC Group that piggybacks a SOCKS proxy onto an active RDP session using Dynamic Virtual Channels (DVC), a built-in feature of the Remote Desktop Protocol. DVCs are the same mechanism Windows uses for clipboard synchronisation and audio redirection between RDP client and server. SocksOverRDP repurposes that channel to carry arbitrary SOCKS proxy traffic without opening any new ports or firewall rules, making it ideal for pure Windows-based assessments where SSH is unavailable.

## How the DVC Mechanism Works

When the SocksOverRDP plugin DLL is registered on the RDP client machine, it hooks into `mstsc.exe` and is loaded every time an RDP session starts. On the server side of the RDP connection, the `SocksOverRDP-Server.exe` executable connects back to the DLL through the virtual channel already established inside the RDP session. Once both sides are active, the plugin spins up a SOCKS5 proxy listener on the client machine at `127.0.0.1:1080`. No new TCP connections or firewall exceptions are required because everything rides inside the existing RDP session.

## Required Files

Two tools are needed before starting:

- `SocksOverRDP-x64.zip` from the [GitHub releases page](https://github.com/nccgroup/SocksOverRDP/releases) -- contains both the plugin DLL and the server EXE
- `ProxifierPE.zip` from [proxifier.com](https://www.proxifier.com/download/#win-tab) -- the portable version that requires no installation

Both should be downloaded to the attack host first, then transferred to targets as needed via RDP clipboard or file copy.

## Step by Step Setup

### Step 1 -- Register the Plugin on the Foothold Host (10.129.x.x)

Transfer `SocksOverRDPx64.zip` to the Windows foothold machine and extract it. Register the plugin DLL so `mstsc.exe` picks it up on the next connection:

```cmd
C:\Users\htb-student\Desktop\SocksOverRDP-x64> regsvr32.exe SocksOverRDP-Plugin.dll
```

A successful registration shows a confirmation dialog. If the user is not a local administrator, the `SocksOverRDP-Plugin.reg` file can be imported instead to write the necessary registry keys under the current user's hive.

### Step 2 -- RDP into the Internal Target (172.16.5.19) and Start the Server

From the foothold machine, open `mstsc.exe` and connect to `172.16.5.19` using the credentials `victor:pass@123`. Because the plugin is registered, a prompt appears confirming it is active and will listen on `127.0.0.1:1080`.

Once inside the RDP session on `172.16.5.19`, transfer `SocksOverRDP-Server.exe` to it and run it with administrator privileges. The server connects back to the plugin on the foothold machine through the DVC, completing the SOCKS channel.

### Step 3 -- Verify the SOCKS Listener

Back on the foothold machine, confirm the SOCKS listener is running:

```cmd
C:\Users\htb-student\Desktop\SocksOverRDP-x64> netstat -antb | findstr 1080

  TCP    127.0.0.1:1080         0.0.0.0:0              LISTENING
```

### Step 4 -- Configure Proxifier and Pivot Further

Transfer the Proxifier portable binary to the foothold machine and configure it:

1. Go to Profile > Proxy Servers and add `127.0.0.1` port `1080` as a SOCKS5 proxy
2. Set it as the default proxy or create a proxification rule targeting all traffic

Once Proxifier is active, launching `mstsc.exe` from the foothold machine and connecting to `172.16.6.155` will route that RDP session's traffic through the SOCKS channel on port 1080, which runs through the existing RDP session to `172.16.5.19`, which then reaches `172.16.6.155` via `SocksOverRDP-Server.exe`. The full traffic path is:

```
Attack Host -> RDP -> 10.129.x.x (foothold) -> SOCKS via DVC -> 172.16.5.19 -> 172.16.6.155
```

## Cleanup

To unregister the plugin after the engagement:

```cmd
regsvr32.exe /u SocksOverRDP-Plugin.dll
```

This removes the DLL from the list of plugins loaded by `mstsc.exe` and reverts the associated registry entries.

## RDP Performance Tuning

Running multiple nested RDP sessions simultaneously over a tunnel can cause noticeable lag. To reduce this, open the Experience tab in `mstsc.exe` before connecting and set the connection speed to Modem. This disables visual features like wallpaper, font smoothing, and window animation that consume significant bandwidth, keeping the session responsive when working through multiple pivot hops.

## When to Use SocksOverRDP

| Condition | SocksOverRDP Suitable |
|---|---|
| Pure Windows environment, no SSH available | Yes |
| RDP access between hosts already established | Yes |
| Linux attack host without Windows tooling | No |
| Need to pivot beyond the first Windows hop | Yes -- chain with Proxifier |
| Only one hop needed | Netsh portproxy is simpler |

SocksOverRDP fills a specific gap: Windows-only environments where every other pivoting technique requires SSH or Linux tooling. The DVC approach is particularly clean because it produces no new listening ports on any firewall-monitored interface.
