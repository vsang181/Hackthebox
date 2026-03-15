# Latest RDP Vulnerabilities

BlueKeep ([CVE-2019-0708](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2019-0708)) is a critical pre-authentication RCE vulnerability in the Windows RDP service published in 2019. Its defining characteristic is that it requires no credentials and no user interaction -- a connection attempt alone is enough to trigger it, making it wormable and comparable in risk profile to EternalBlue.

## The Vulnerability

BlueKeep is classified as a [Use-After-Free (UAF)](https://cwe.mitre.org/data/definitions/416.html) vulnerability. A UAF occurs when a program continues to reference a block of memory after that memory has been freed. The attacker manipulates the initial settings exchange between client and server -- before any authentication takes place -- to trigger a vulnerable function responsible for creating a virtual channel. Once that function is exploited and the memory is freed, the attacker writes a payload into the freed kernel memory. Because the CPU has been redirected to execute from that location, the attacker's instructions run with the full privileges of the kernel process.

A detailed technical breakdown of the three methods used to write data into the kernel via RDP PDUs is available from [Palo Alto's Unit42 analysis](https://unit42.paloaltonetworks.com/exploitation-of-windows-cve-2019-0708-bluekeep-three-ways-to-write-data-into-the-kernel-with-rdp-pdu/).

## Mapping to the Concept of Attacks

The exploit runs through two complete cycles.

### Cycle 1 -- Initiating the Attack

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker sends a manipulated initialisation request during the RDP settings exchange, before authentication | Source |
| 2 | The request triggers a vulnerable function that creates a virtual channel | Process |
| 3 | The RDP service runs under the LocalSystem Account, the highest privilege level on a Windows system | Privileges |
| 4 | The manipulated function redirects execution flow to a kernel process | Destination |

### Cycle 2 -- Triggering Remote Code Execution

| Step | What Happens | Category |
|---|---|---|
| 5 | The attacker's payload is inserted into the process to free kernel memory and position arbitrary instructions | Source |
| 6 | The kernel process frees the targeted memory and the CPU is directed to the attacker's code | Process |
| 7 | Execution occurs under LocalSystem Account privileges, inherited from the kernel context | Privileges |
| 8 | A reverse shell is sent back over the network to the attacker's host | Destination |

## Affected Systems and Current State

BlueKeep affects Windows 7, Windows Server 2008, and Windows Server 2008 R2. Windows 8 and Windows 10 onward are not vulnerable. Microsoft released patches for affected versions, including out-of-band patches for Windows XP and Server 2003, which were no longer in mainstream support at the time. Despite this, approximately 950,000 systems were identified as vulnerable in an initial scan in May 2019, and roughly a quarter of those remained unpatched in subsequent surveys.

The vulnerability is particularly relevant in environments running legacy systems that cannot be easily updated -- healthcare organisations, industrial control environments, and any infrastructure where software is locked to a specific OS version for compatibility reasons are consistently at risk.

## Operational Caution

BlueKeep is one of the few vulnerabilities in this module that carries a meaningful risk of crashing the target system. Exploitation can produce a Blue Screen of Death (BSoD) if the memory conditions are not met precisely, which makes it inherently more disruptive than credential-based attacks. Before running an exploit for this vulnerability during an engagement, the recommendation is to discuss the risk explicitly with the client so they can make an informed decision about whether to proceed. Confirming the vulnerability exists via Nmap's `rdp-vuln-ms12-020` script or a dedicated BlueKeep detection tool provides the evidence needed without the crash risk of a full exploit attempt.
