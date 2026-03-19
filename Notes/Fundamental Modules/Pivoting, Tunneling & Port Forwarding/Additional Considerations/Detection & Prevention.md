# Detection & Prevention

Detection and prevention in the context of pivoting and tunneling is not just about deploying technical controls -- it requires a structured approach across people, processes, and technology working together. A defender who only focuses on blocking tools will always be a step behind an attacker who can live off the land and blend into normal traffic.

## Setting a Baseline First

You cannot detect what is abnormal if you have no definition of normal. Before implementing any detection capability, document everything that currently exists and operates in the environment. The most useful items to track and audit regularly include:

- DNS records, DHCP configurations, and network device backups
- A full and current application inventory including versions
- A list of all enterprise hosts, their locations, and their network interfaces
- Users with elevated privileges and which systems they can access
- Dual-homed hosts with more than one network interface, as these are natural pivot candidates
- A visual and up-to-date network diagram showing all segments and trust relationships

Tools like [Netbrain](https://www.netbraintech.com/) provide interactive, live network diagrams, while [diagrams.net](https://app.diagrams.net/) offers a free option for smaller environments. The goal is that when something appears that should not be there -- a new host, a new listener, unusual traffic -- it stands out immediately against the documented baseline.

## People

Users remain the most consistently exploited entry point into enterprise networks. The BYOD scenario in the material illustrates this clearly: a single employee's poor security habits on a personal device extended an attacker's reach directly into the corporate network simply because that device was allowed to connect to the employee WiFi segment. A few foundational controls around the human element:

- Multi-factor authentication for all remote access, and especially for any administrative or privileged account -- even a compromised password becomes significantly less useful when MFA is enforced
- Clear and enforced BYOD policies that define what personal devices are permitted to access, which network segments, and under what conditions
- Regular security awareness training covering phishing, social engineering, and safe browsing -- these attacks are the most common initial access vector
- A Security Operations Center (SOC) or a contracted SOC-as-a-Service to ensure human eyes are reviewing alerts and logs around the clock, since automated controls alone cannot handle incident response

## Processes

Without defined and documented procedures, accountability is nearly impossible and incident response becomes chaotic under pressure. The processes that matter most for defending against the techniques covered in this module are:

- Access control policies covering account provisioning, deprovisioning, and the principle of least privilege -- accounts should only have access to what they need and nothing beyond that
- Change management documentation so that every configuration change has a record of who made it, when, and why -- this is essential for separating legitimate administrative activity from attacker activity
- Host provisioning and decommissioning procedures built around a hardened gold image baseline, so every new system enters the network in a known-good state
- A tested and practiced disaster recovery and incident response plan -- having one on paper that has never been exercised provides minimal real-world value

## Technology

Technology controls should be layered from the perimeter inward, with the understanding that a perimeter breach is a matter of when, not if.

At the perimeter, a next-generation firewall with the capability to block suspicious connections by IP, enforce VPN access policies, and quickly sever suspicious sessions is the first layer. Strict egress filtering that limits which protocols and ports are allowed outbound is particularly relevant to the tunneling techniques covered in this module -- protocols like ICMP, DNS, and HTTP being allowed out indiscriminately are exactly what tools like ptunnel-ng, dnscat2, and Chisel rely on.

Internally, network segmentation prevents lateral movement from escalating into full compromise. Keeping HR users on a separate segment from infrastructure devices, isolating production from management networks, and placing internet-facing hosts in a properly configured DMZ limits what an attacker can reach even after they have a foothold.

A well-configured SIEM that ingests logs from endpoints, firewalls, DNS servers, and network devices gives defenders the visibility needed to correlate events across the environment. A single log source in isolation rarely tells the full story. Combined log sources can reveal patterns that individual alerts would miss.

## MITRE ATT&CK Mapping

The techniques practiced throughout this module map directly to documented adversary behaviors. Understanding these mappings helps defenders prioritize controls and configure detection rules that align with known attacker patterns.

| TTP | MITRE Tag | Key Defensive Controls |
|---|---|---|
| External Remote Services | T1133 | Perimeter firewall, VPN enforcement, block unneeded inbound protocols |
| Remote Services (SSH/RDP) | T1021 | MFA, least-privilege access, host and network firewall rules, OOB management for infrastructure |
| Non-Standard Ports | T1571 | Protocol-to-port baseline monitoring, NIDS/NIPS rules flagging protocol mismatches |
| Protocol Tunneling | T1572 | Restrict allowed protocols outbound, DNS resolution controlled internally, C2 beaconing detection via traffic pattern analysis |
| Proxy Use | T1090 | Allowlist-based domain and IP filtering, net flow analysis, monitoring for indirect traffic paths |
| Living Off the Land (LOTL) | N/A | Behavioral baseline, EDR with shell execution monitoring, SIEM correlation of user and process anomalies |

## Protocol Tunneling Detection in Practice

Protocol tunneling deserves particular attention because it is the thread connecting most of the tools covered in this module. DNS tunneling, ICMP tunneling, HTTP-wrapped SSH via Chisel, and SOCKS-over-RDP all share a common characteristic: they carry payload data inside a protocol that is normally permitted through firewalls without deep inspection.

Detection strategies that work across these techniques include:

- Monitoring DNS for high query volumes, high-entropy subdomain names, or oversized TXT record payloads -- all indicators of dnscat2-style tunneling
- Flagging ICMP packets with unusually large data payloads or abnormally high frequency, which distinguishes ptunnel-ng traffic from legitimate ping usage
- Watching for beaconing behavior, which is regular, periodic outbound traffic at predictable intervals -- this is a common property of C2 channels regardless of the protocol being used to carry them
- Alerting on non-standard port and protocol pairings, such as SSH traffic on port 8080 or HTTP traffic on port 53

No single control stops all of these. Layering network-based inspection, endpoint visibility through EDR, and centralized log correlation through a SIEM is what gives defenders the best realistic chance of catching a pivoting attacker before they reach their target.
