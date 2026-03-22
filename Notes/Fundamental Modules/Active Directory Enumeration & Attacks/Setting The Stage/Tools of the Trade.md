# Tools of the Trade

The AD attack toolkit is large, and knowing which tool to reach for in which situation matters as much as knowing how each one works. The tools used across this module can be grouped by what stage of an engagement they serve: initial enumeration and mapping, credential attacks, lateral movement and execution, and exploitation of specific AD features.
## Enumeration and Mapping
These tools build the picture of the AD environment before any active attacks are launched.

**PowerView / SharpView** are the most flexible manual enumeration tools available. PowerView is a PowerShell script and SharpView is its compiled .NET equivalent, both from the PowerSploit framework. They replicate and extend the functionality of native Windows `net *` commands, letting you query users, groups, computers, ACLs, trusts, and GPOs with fine-grained control. They are particularly useful for checking what a newly obtained set of credentials can access, or for quickly identifying Kerberoastable and ASREPRoastable accounts without running a full BloodHound collection.

**BloodHound** is the graph-based AD attack path visualiser. It ingests JSON data collected by SharpHound or BloodHound.py and maps every relationship between users, groups, computers, and GPOs into a Neo4j database. The key value is the Cypher query engine, which can answer questions like "find the shortest path from this domain user to Domain Admin" in seconds, surfacing attack paths that manual enumeration would take hours to piece together. The `--Throttle` and `--Jitter` options in SharpHound allow operators to slow collection speed to reduce detection noise.

**SharpHound** is the C# ingestor that feeds BloodHound. It collects users, groups, computers, sessions, ACLs, GPOs, and trust information and outputs JSON files for import. It runs on the target or from a domain-joined host.

**BloodHound.py** is the Python-based alternative ingestor from the Impacket ecosystem. It does not require a domain-joined Windows host and can run directly from a Linux attack machine with domain credentials, making it the preferred option when working from an external Linux host.

**ADRecon** produces an Excel-formatted report of the entire AD environment including users, computers, trusts, GPOs, and security configurations, useful for producing a structured snapshot for later offline analysis.

**Active Directory Explorer** from Sysinternals is a GUI-based AD browser that allows navigation of the directory tree, inspection of object attributes, and saving offline snapshots for comparison over time. Comparing two snapshots is useful for identifying configuration changes between assessment stages.

**PingCastle** audits the security posture of an AD environment against a risk and maturity framework, scoring the domain and surfacing common misconfigurations in a structured report format.

**Group3r** focuses specifically on Group Policy Objects, auditing them for security misconfigurations that could be exploited for privilege escalation or persistence.

**adidnsdump** enumerates and dumps DNS records from Active Directory, similar in result to a DNS zone transfer, useful for mapping internal host naming and identifying infrastructure.
## LDAP and RPC Enumeration
**ldapsearch** is the built-in Linux command-line tool for querying LDAP directly. It requires knowing the right LDAP filter syntax but provides raw access to any attribute stored in the directory.

**windapsearch** wraps common LDAP queries into a Python script, automating enumeration of users, groups, computers, and privileged accounts without needing to write raw filters manually.

**enum4linux / enum4linux-ng** enumerate SMB-exposed information from Windows and Samba hosts including user lists, share listings, password policies, and domain information. The `-ng` variant is a rewritten version with better output formatting and additional features.

**rpcclient** from the Samba suite provides an interactive shell for sending RPC calls to Windows hosts, useful for enumerating users, groups, and shares over the SMB/RPC channel.

**rpcinfo** queries the status of RPC services on a remote host, listing available programs, version numbers, and protocols.

**rpcdump.py** from Impacket enumerates RPC endpoints exposed on a remote host.

**smbmap** enumerates SMB shares across domain hosts, listing accessible shares and permissions for a given set of credentials.
## Credential Attacks
**Kerbrute** is a Go-based tool that uses Kerberos pre-authentication to enumerate valid domain accounts and perform password spraying without triggering the standard LDAP failed-login counters. Because it operates at the Kerberos layer rather than through LDAP bind attempts, it is significantly stealthier than SMB-based spraying.

**DomainPasswordSpray.ps1** performs password spraying from within a domain-joined context using PowerShell. Its key operational advantage is that it reads the domain password policy and automatically removes accounts close to lockout from the target list before each spray round.

**GetNPUsers.py** (Impacket) performs ASREPRoasting, identifying and requesting AS-REP hashes for accounts that have Kerberos pre-authentication disabled. These hashes can then be cracked offline with Hashcat.

**GetUserSPNs.py** (Impacket) performs Kerberoasting, requesting TGS service tickets for accounts that have SPNs registered and outputting them in a format ready for Hashcat cracking.

**Hashcat** handles offline cracking of captured hashes including NTLM, NetNTLMv2, Kerberos TGS, and AS-REP formats. Combined with rules like d3ad0ne and large wordlists, it is the primary tool for converting captured hashes to plaintext credentials.

**gpp-decrypt** extracts credentials embedded in Group Policy Preferences files (Groups.xml and similar), which in older environments contain AES-encrypted passwords that Microsoft publicly disclosed the key for.
## Credential Capture and Relay
**Responder** poisons LLMNR, NBT-NS, and MDNS queries on the local network segment to capture NetNTLMv2 hashes from Windows hosts that attempt to resolve hostnames that do not exist in DNS. It also spins up rogue SMB, HTTP, FTP, and other protocol servers to receive authentication attempts.

**Inveigh.ps1 / InveighZero** perform the same spoofing and poisoning functions as Responder but run on Windows hosts. InveighZero is the C# version with an interactive console for browsing captured credentials in real time, useful when the attack host is a Windows machine.

**ntlmrelayx.py** (Impacket) relays captured NTLM authentication attempts to other services rather than cracking them. If a captured hash belongs to a user with local admin rights on another host, ntlmrelayx can relay that authentication to dump SAM/LSA secrets, execute commands, or establish SOCKS connections for further access.
## Lateral Movement and Execution
**psexec.py** (Impacket) provides a semi-interactive shell on a remote host using the same mechanism as Sysinternals PsExec, authenticating over SMB and dropping a service binary for execution. Requires admin privileges on the target.

**wmiexec.py** (Impacket) executes commands on remote hosts over WMI, producing output without writing a binary to disk. Slightly stealthier than psexec.py because it does not create a service.

**evil-winrm** provides an interactive shell over WinRM (port 5985/5986) with built-in features for file upload, download, and PowerShell script loading. The preferred remote shell option when WinRM is accessible.

**mssqlclient.py** (Impacket) interacts with MSSQL servers, supporting authentication via Windows credentials, NTLM hashes, or Kerberos tickets, and enabling command execution through `xp_cmdshell`.

**smbserver.py** (Impacket) stands up a quick SMB share on the attack host for transferring tools and files to and from Windows targets without needing a full file server.

**Mimikatz** is the most well-known credential extraction tool. Its primary functions in AD engagements are dumping plaintext credentials and NTLM hashes from LSASS memory, performing pass-the-hash attacks, extracting Kerberos tickets from memory for pass-the-ticket attacks, and running DCSync to replicate domain hashes.

**secretsdump.py** (Impacket) remotely dumps SAM, LSA, and NTDS.dit hashes from a host using DCSync or registry-based extraction. It does not require dropping Mimikatz on the target and can run entirely from the Linux attack host.

**Rubeus** is a C# tool dedicated to Kerberos abuse. It handles Kerberoasting, ASREPRoasting, TGT and TGS extraction from memory, pass-the-ticket, ticket renewal, and S4U delegation abuse all from within a Windows context.
## Kerberos Ticket and Certificate Manipulation
**ticketer.py** (Impacket) creates and customises TGT and TGS tickets, primarily used for Golden Ticket and Silver Ticket attacks and cross-domain trust exploitation.

**gettgtpkinit.py** manipulates PKINIT certificates and TGTs, used in Shadow Credentials and AD CS-based attacks.

**getnthash.py** uses an existing TGT to request a PAC via User-to-User (U2U) authentication, extracting the NT hash of the account without needing to crack anything.

**lookupsid.py** (Impacket) performs SID brute-forcing to enumerate domain and local accounts by iterating through RID values, useful when other enumeration methods are restricted.

**raiseChild.py** (Impacket) automates child-to-parent domain privilege escalation by forging an inter-realm trust ticket.

**setspn.exe** is a native Windows tool for reading, adding, and modifying Service Principal Names on AD accounts, used when creating or cleaning up Kerberoastable service accounts during testing.
## Specific Exploit Tools
**noPac.py** exploits the combination of CVE-2021-42278 and CVE-2021-42287 to allow any standard domain user to impersonate a Domain Admin by manipulating machine account naming.

**CVE-2021-1675.py** is a PrintNightmare PoC that exploits the Windows Print Spooler service to achieve remote code execution and local privilege escalation.

**PetitPotam.py** exploits CVE-2021-36942 to coerce Windows hosts into authenticating to an attacker-controlled machine via the MS-EFSRPC interface, triggering NTLM relay or Kerberos authentication capture.
## Post-Compromise Auditing
**Snaffler** crawls accessible file shares across the domain looking for sensitive data including credentials, configuration files, private keys, and connection strings. It runs from a domain-joined context and prioritises findings by sensitivity level.

**LAPSToolkit** audits environments running Microsoft's Local Administrator Password Solution, identifying which computers have LAPS deployed, which accounts can read LAPS passwords, and retrieving those passwords where access permits.
