## Incident Handling - Definition and Scope

Incident handling is a clearly defined set of procedures for managing and responding to security incidents in a computer or network environment. Before examining the process itself, two foundational terms must be understood precisely, as the distinction matters throughout every investigation.

- An **event** is any action occurring in a system or network: a user sending an email, a firewall allowing a connection, a mouse click
- An **incident** is an event with a negative consequence and, in the IT security context specifically, a clear intent to cause harm against a computer system

Incidents include data theft, funds theft, unauthorized access, malware installation, and insider misuse. Importantly, incident handling is not limited to intrusions. Availability disruptions, intellectual property loss, and malicious insider activity all fall within its scope.

***

## Why Incident Handling Matters

In 2024, 86% of major cyber incidents resulted in operational downtime, reputational damage, or financial loss, based on over 500 IR engagements across 38 countries in the [Unit 42 Global Incident Response Report](https://www.paloaltonetworks.com/engage/unit42-2025-global-incident-response-report).  The 2026 edition of the same report found that the fastest 25% of intrusions reached data exfiltration in just 72 minutes, down from 285 minutes the previous year.  These numbers illustrate why a trained, structured incident handling capability is not optional for any organization that depends on the confidentiality, integrity, or availability of its data. 

The objective of an incident response team is to minimise the theft of information or disruption of services. Every decision made before, during, and after an incident shapes its ultimate impact. The team is led by an **incident manager** who acts as the single point of communication, tracks all actions taken, and has the authority to direct other business units. This role is typically held by a SOC manager, CISO, or trusted third-party IR provider.

The primary reference standard for incident handling is [NIST SP 800-61r3](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-61r3.pdf), updated in April 2025 to align with the NIST Cybersecurity Framework (CSF) 2.0.  The revised publication introduces a more flexible, continuous lifecycle model and emphasises integrating incident response into broader cybersecurity risk management rather than treating it as an isolated reactive function.

***

## Real-World Incident Categories

Understanding recurring incident types helps analysts recognise patterns quickly during triage.

### Leaked or Weak Credentials

The [Colonial Pipeline ransomware attack](https://en.wikipedia.org/wiki/Colonial_Pipeline_ransomware_attack) began with a single compromised password for an inactive VPN account that lacked MFA. The [Mirai botnet](https://en.wikipedia.org/wiki/Mirai_(malware)) in 2016 compromised hundreds of thousands of IoT devices using factory default credentials like `admin/admin`, driving large-scale DDoS attacks against Dyn and OVH. The 2023 LogicMonitor incident involved vendor-assigned weak default passwords that left customers exposed to ransomware. All three share a root cause: credential hygiene failures that are cheap to prevent but catastrophic when ignored.

### Unpatched Systems

The [Equifax breach](https://en.wikipedia.org/wiki/2017_Equifax_data_breach) in 2017 exposed the personal data of approximately 147 million people through a known Apache Struts vulnerability (CVE-2017-5638) that had a patch available but was not applied. [WannaCry](https://en.wikipedia.org/wiki/WannaCry_ransomware_attack) spread as a worm via the EternalBlue SMB exploit across more than 200,000 systems in 150 countries, despite the MS17-010 patch being available before the outbreak. Both incidents confirm that patch management delay is a primary enabler of large-scale compromise.

### Insider Threats

The Cash App / Block Inc. incident involved a former employee accessing personal data belonging to approximately 8.2 million customers, enabled by insufficient internal access controls and monitoring after employment ended. In 2025, insider threat cases involving North Korean operatives tripled, with attackers targeting contract-based technical roles at tech firms, financial services companies, and government defence contractors. 

### Phishing and Social Engineering

Social engineering remained the top initial access vector in Unit 42 IR cases between May 2024 and May 2025, accounting for 36% of all incidents, and more than a third of those involved non-phishing techniques such as SEO poisoning, fake system prompts, and help desk manipulation.  The 2020 Twitter account hijacking, in which attackers used social engineering against Twitter employees to gain access to internal administrative tools and compromise high-profile accounts for a bitcoin scam, is a well-documented example of how social engineering bypasses technical controls entirely by targeting human trust. 

### Supply Chain Attacks

The [SolarWinds Orion](https://en.wikipedia.org/wiki/2020_United_States_federal_government_data_breach) compromise in 2020 involved nation-state actors injecting a malicious backdoor into legitimate software updates distributed to thousands of customers across government and private sectors. In 2024, attackers scanned more than 230 million unique targets in a single supply chain and cloud campaign, illustrating the scale modern supply chain attacks can achieve. 

***

## Incident Report Structure

Professional incident reports follow a sequential, stage-by-stage format aligned with the Cyber Kill Chain or MITRE ATT&CK framework, moving from initial access through to impact. Two types of reports are commonly produced:

- **Incident-specific reports** (such as the [Confluence Exploit Leads to LockBit Ransomware](https://thedfirreport.com/2025/02/24/confluence-exploit-leads-to-lockbit-ransomware/) from DFIR Report) narrate a single attack forensically with step-by-step technical findings and remediation actions
- **Global trend reports** (such as the [Unit 42 Global Incident Response Report](https://www.paloaltonetworks.com/engage/unit42-2025-global-incident-response-report)) aggregate data from hundreds of incidents to surface attacker trends, emerging techniques, and strategic recommendations for defenders

Other respected sources for incident reports include [Mandiant](https://www.mandiant.com/), [Palo Alto Unit 42](https://unit42.paloaltonetworks.com/), and [Proofpoint](https://www.proofpoint.com/).

***

## Module Scenario - Insight Nexus

Throughout this module, the reference scenario involves **Insight Nexus**, a global market research firm handling sensitive competitive intelligence for high-profile clients. Two threat groups are operating simultaneously within its environment.

The first threat actor exploited a default `admin/admin` credential left unchanged on an internet-facing ManageEngine ADManager Plus instance after a product update. From there, the attacker:

1. Logged in and performed internal reconnaissance
2. Mapped domain users and machines
3. Created new privileged Active Directory accounts
4. Pivoted to an exposed RDP service left open by misconfiguration
5. Used Group Policy Objects (GPOs) to deploy spyware via an MSI package across multiple endpoints

This scenario mirrors patterns seen repeatedly in real-world incidents and demonstrates how a single credential oversight cascades into full domain compromise. The next sections map this attacker lifecycle against the Cyber Kill Chain and MITRE ATT&CK frameworks.
