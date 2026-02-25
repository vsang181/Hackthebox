# Detection & Prevention

Effective detection and prevention of shell-based attacks requires visibility across three distinct layers: network traffic, host activity, and application behaviour. The [MITRE ATT&CK Framework](https://attack.mitre.org/) provides the structured vocabulary for mapping observed activity to known adversary techniques and is the most widely adopted reference for building detection logic against the techniques covered throughout this module.

## MITRE ATT&CK Mapping

Three tactics and techniques are directly relevant to the shell and payload techniques covered in this module:

| Tactic / Technique | ATT&CK Reference | Relevance to Shells and Payloads |
|---|---|---|
| **Initial Access** | [T1190: Exploit Public-Facing Application](https://attack.mitre.org/techniques/T1190/) | Gaining a foothold via vulnerable web applications, misconfigured services such as SMB, or exposed authentication portals |
| **Execution** | [T1059: Command and Scripting Interpreter](https://attack.mitre.org/techniques/T1059/) | Running payloads and shell commands via PowerShell, Bash, cmd.exe, and other interpreters; covers web shell command execution, one-liners, and MSF-delivered stages |
| **Command and Control** | [TA0011: Command and Control](https://attack.mitre.org/tactics/TA0011/) | Maintaining interactive access via reverse shells, Meterpreter sessions, and C2 frameworks communicating over standard ports and protocols |

## Events to Watch For

Identifying active shells and payload execution relies on correlating events across multiple data sources. Key indicators to monitor include:

- **File uploads to web applications:** Application logs should be reviewed for uploads of unexpected file types, particularly script extensions (`.php`, `.aspx`, `.jsp`) being written to web-accessible directories. Any file upload function exposed to the internet should have both server-side type validation and post-upload file integrity monitoring.
- **Anomalous command execution by non-administrative accounts:** Standard users have no legitimate reason to run commands such as `whoami`, `net user`, `id`, or `ipconfig` interactively. Enabling PowerShell ScriptBlock logging, module logging, and Windows command-line auditing (via Group Policy or Sysmon) will surface these events. Windows Event ID **4688** with command-line auditing captures process creation with full argument strings.
- **Unexpected SMB connections between end hosts:** Lateral movement frequently involves SMB connections from workstation to workstation rather than from workstation to server. This pattern deviates from normal infrastructure traffic flow and warrants investigation.
- **Anomalous network sessions:** Users and servers follow predictable traffic patterns. Deviations worth flagging include persistent TCP connections to external IPs on non-standard ports (particularly port 4444, the Metasploit default), periodic low-volume outbound connections that match a beaconing interval, bulk GET or POST requests in short windows, and DNS queries to newly registered or algorithmically generated domains.

## Establishing Network Visibility

Network-level visibility is foundational to detecting shell callbacks and payload delivery. An unencrypted reverse shell or bind shell session transmits all commands and output in cleartext, making it trivially readable to anyone capturing traffic on the path. The following example illustrates this: NetFlow data showing sustained TCP traffic between two hosts on port 4444 is immediately suspicious, and expanding the capture reveals `net user` commands being issued to create a backdoor account on the target host.

Tools and practices that establish the network baseline needed to make this detection possible:

- **Network topology documentation:** Maintaining current visual network diagrams using tools such as [Draw.io](https://draw.io/) or enterprise platforms such as [NetBrain](https://www.netbraintech.com/) ensures that defenders understand expected traffic flows and can identify deviations.
- **Cloud-managed network controllers:** Vendors including Cisco Meraki, Palo Alto Networks, and Ubiquiti provide Layer 7 visibility dashboards through cloud-based management planes, offering application-level traffic classification and anomaly alerting without requiring on-premises appliances.
- **[Deep packet inspection (DPI)](https://en.wikipedia.org/wiki/Deep_packet_inspection):** Network security appliances capable of inspecting packet payloads beyond the IP header and port number can detect shell traffic even when it runs on common ports such as 443. Unencrypted Netcat sessions are particularly visible to DPI; encrypted Meterpreter traffic over TLS is harder to inspect but can still be flagged through certificate analysis and traffic pattern anomalies.

## Protecting End Devices

End devices, whether workstations, servers, printers, or network-attached storage, are the systems that shells ultimately execute on. Prioritising their protection limits the impact of a successful initial access event. Key controls include:

- **Antivirus and endpoint protection:** On Windows, **[Windows Defender](https://www.microsoft.com/en-gb/windows/comprehensive-security)** should remain enabled at all times with real-time protection active. Its AMSI integration intercepts malicious PowerShell payloads at the script block level before execution. AV on servers introduces a performance overhead but provides a critical last line of defence against payload execution.
- **Windows Firewall:** All three profiles (Domain, Private, and Public) should remain active. Exceptions should only be created through a formal change management process and should be as specific as possible, scoping to approved application binaries rather than open port ranges.
- **Patch management:** The majority of high-impact Windows exploits covered in this module, including EternalBlue and PrintNightmare, were exploitable because organisations failed to apply available patches promptly. A structured [patch management](https://www.rapid7.com/fundamentals/patch-management/) process reduces this window of exposure.

## Potential Mitigations

| Mitigation | Description |
|---|---|
| **Application sandboxing** | Isolate public-facing applications to limit the scope of access an attacker gains if a vulnerability is exploited; compromise of the application should not equate to compromise of the underlying host or network |
| **Least privilege** | Users and service accounts should hold only the permissions required for their function; the `apache` user obtaining `NOPASSWD: ALL` sudo rights, as seen in this module, is a direct result of misconfigured privilege assignment |
| **Host segmentation and hardening** | Place internet-facing hosts such as web servers, VPN concentrators, and mail relays in a DMZ or dedicated quarantine segment; apply STIG hardening guidelines to reduce the attack surface of each host before exposure |
| **Physical and application layer firewalls** | Strictly scoped inbound and outbound rules that permit only traffic initiated from within the network, on approved ports, and to approved destinations will block the majority of bind and reverse shell connections; NAT implementations further disrupt inbound bind shell connectivity |

A defence-in-depth approach, combining network visibility, host-level endpoint protection, least privilege enforcement, segmentation, and active monitoring, is the most resilient posture against the techniques covered in this module. No single control is sufficient in isolation; layered defences ensure that the failure of any one control does not result in a complete compromise.
