# SSH for Windows: plink.exe

[Plink](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html) is the command-line SSH companion to the PuTTY terminal client, and it brings SSH tunneling capability to Windows hosts without requiring a full Linux environment. Before Windows 10's October 2018 update introduced a native OpenSSH client, PuTTY was one of the most common SSH clients installed by Windows sysadmins, making plink a realistic tool to find already present on a compromised Windows host.

## When Plink Is Useful

The primary appeal of plink in a red team context is that it is a legitimate, widely trusted tool. In a moderately locked-down environment where uploading custom binaries risks triggering endpoint detection, finding plink already installed -- or locating a copy on a file share -- provides a ready-made SSH tunneling capability that blends naturally with legitimate administrative activity. It is a textbook example of living off the land.

Two scenarios where plink becomes the right choice:

- A compromised Windows host needs to serve as a pivot point and SSH tooling is already present
- The attack host is Windows-based rather than Linux, and standard SSH commands are unavailable or inconvenient

## Creating a Dynamic Port Forward with Plink

The syntax mirrors the SSH `-D` flag covered in the dynamic port forwarding section. From a CMD session on the Windows host:

```cmd
plink -ssh -D 9050 ubuntu@10.129.15.50
```

This establishes an SSH session from the Windows host to the Ubuntu pivot server and starts a SOCKS listener on `127.0.0.1:9050`. All traffic directed to that port is forwarded through the SSH tunnel into whatever network the Ubuntu server can reach.

On first connection to a new host, plink will prompt to accept and cache the host key. In automated or non-interactive scenarios, this can be suppressed by prepending `echo y |` to auto-accept:

```cmd
echo y | plink -ssh -D 9050 ubuntu@10.129.15.50
```

## Routing Windows Application Traffic Through the SOCKS Proxy

On Linux, proxychains intercepts tool traffic and routes it through SOCKS. On Windows, the equivalent is [Proxifier](https://www.proxifier.com/). Proxifier is a commercial tool that forces any Windows application's TCP traffic through a configured SOCKS or HTTPS proxy, including applications that have no built-in proxy support.

The configuration is straightforward:

1. Open Proxifier and go to Profile > Proxy Servers
2. Add a new proxy: address `127.0.0.1`, port `9050`, protocol SOCKS4 or SOCKS5 to match the plink session
3. Set the proxy as default or create a proxification rule targeting specific applications

Once Proxifier is active with the SOCKS proxy configured, launching `mstsc.exe` (the native Windows RDP client) and connecting to an internal IP like `172.16.5.19` will route the RDP traffic through the plink tunnel. The RDP client itself requires no configuration changes -- Proxifier intercepts the traffic transparently at the network layer.

## Plink vs Native SSH on Windows

Since Windows 10 1809, a native OpenSSH client ships with Windows and the same `-D`, `-L`, and `-R` flags used on Linux work identically. Plink remains relevant in two specific situations: older Windows versions that predate the native client, and environments where PuTTY is already installed and trusted while running custom binaries would be flagged. Knowing both options means there is always a path forward regardless of the Windows version on the compromised host.
