## Information Gathering

Information gathering is the most recurring phase across the entire penetration testing process. Every exploitation step, every privilege escalation path, and every lateral movement decision is built on the quality of information collected here. It divides into four distinct categories that apply regardless of whether you are working externally or internally.

***

## The Four Categories

```
1. Open-Source Intelligence (OSINT)    <- before touching a single system
2. Infrastructure Enumeration          <- map the perimeter and internal structure
3. Service Enumeration                 <- identify what is running and what version
4. Host Enumeration                    <- deep-dive each individual target
```

All four must be performed on every engagement. Skipping any one of them creates blind spots that will cost you findings in the report and access in the field.

***

## 1. Open-Source Intelligence (OSINT)

Passive reconnaissance using only publicly available sources. No direct interaction with client systems at this stage.

```
Company-level OSINT targets:
- Job postings (reveal tech stack, internal tools, security gaps)
- LinkedIn (employee names, roles, seniority, org structure)
- Company website (technologies used, subdomains, staff names)
- Press releases and news (recent acquisitions, new platforms)
- WHOIS and domain registration records
- Certificate transparency logs (subdomains via crt.sh)
- Shodan / Censys (internet-exposed services and banners)
- Google dorking:
    site:target.com filetype:pdf
    site:target.com inurl:admin
    site:target.com "index of"
    "target.com" filetype:xlsx OR filetype:docx
    site:github.com "target.com" password OR secret OR key

Code repository OSINT (high-value targets):
- GitHub, GitLab, Bitbucket
- Search: org:companyname password
- Search: org:companyname secret_key
- Search: org:companyname BEGIN RSA PRIVATE KEY
- Search: org:companyname api_key OR token OR credential
- Check commit history - developers often delete secrets but commit history remains
- truffleHog, gitleaks for automated scanning of repos

StackOverflow / Pastebin / developer forums:
- Developers posting code snippets that include real credentials
- Error messages that reveal internal hostnames, paths, software versions
- Database connection strings posted for debugging help

Email OSINT:
- hunter.io, phonebook.cz for email format discovery
- theHarvester for email and subdomain enumeration
- LinkedIn for naming convention inference (first.last@company.com)

Critical finding protocol:
If you find exposed credentials, SSH keys, or tokens during OSINT:
1. Stop and refer to the Incident Handling section of the RoE
2. Notify the client emergency contact before proceeding
3. Document the finding with timestamp and source URL
4. Client must assess and remediate before you continue
```

***

## 2. Infrastructure Enumeration

Build a complete map of the target's internet and internal footprint.

```
External Infrastructure:
DNS enumeration:
  dnsenum --dnsserver <ns> --enum -p 0 -s 0 -o subdomains.txt -f dns-wordlist.txt target.com
  dig any target.com @<nameserver>
  fierce --domain target.com
  subfinder -d target.com -silent
  amass enum -passive -d target.com

Zone transfer attempt (misconfigured DNS):
  dig axfr target.com @<nameserver>
  # If successful: full internal hostname list returned

Certificate transparency:
  curl -s "https://crt.sh/?q=%25.target.com&output=json" | jq '.[].name_value' | sort -u

IP range mapping:
  # Identify ASN from company name
  whois -h whois.radb.net -- '-i origin AS<number>'
  # Returns all CIDR ranges registered to that AS

Cloud footprint:
  # S3 bucket enumeration
  s3scanner scan --buckets-file company-names.txt
  # Azure blob: companyname.blob.core.windows.net
  # GCP: storage.googleapis.com/companyname

Internal Infrastructure (post-foothold):
  # Identify internal DNS servers
  cat /etc/resolv.conf
  ipconfig /all (Windows)

  # Internal subnet discovery
  arp -a
  route print / ip route show
  netdiscover -r 10.10.10.0/24

  # Map internal hosts for password spraying targets
  nmap -sn 10.10.10.0/24 -oA internal_hosts

Security measure identification:
  # WAF detection
  wafw00f https://target.com
  # Load balancer detection
  lbd target.com
  # CDN detection (impacts IP-based scanning)
  Check: dig target.com - if returns Cloudflare/Akamai IP, real IP is hidden
```

***

## 3. Service Enumeration

Identify services, versions, and banners on all discovered hosts.

```
Nmap scan progression:

# Stage 1: Fast discovery sweep (all hosts)
nmap -sn 10.10.10.0/24 -oA discovery

# Stage 2: Top 1000 ports on all live hosts
nmap -sV -sC --open -oA top1000 10.10.10.0/24

# Stage 3: Full port scan on interesting hosts
nmap -p- --min-rate 5000 -oA fullscan <host>

# Stage 4: Targeted version + script scan on open ports
nmap -p 22,80,443,445,3389 -sV -sC -A -oA targeted <host>

# Stage 5: UDP scan (often missed, contains SNMP, TFTP, NTP)
nmap -sU --top-ports 100 -oA udp_scan <host>

Key services and what to extract:

Port 21 (FTP):
  nmap -p 21 --script ftp-anon,ftp-bounce,ftp-syst,ftp-vsftpd-backdoor <host>
  # Check: anonymous login, banner version, directory listing

Port 22 (SSH):
  # Banner reveals OS and SSH version
  # Check for weak key algorithms:
  nmap -p 22 --script ssh2-enum-algos <host>

Port 25/465/587 (SMTP):
  nmap -p 25 --script smtp-commands,smtp-enum-users <host>
  # User enumeration via VRFY / EXPN / RCPT TO

Port 53 (DNS):
  nmap -p 53 --script dns-recursion,dns-zone-transfer <host>

Port 80/443 (HTTP/HTTPS):
  whatweb -v https://target.com
  nikto -h https://target.com
  gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/big.txt -x php,html,txt
  ffuf -w wordlist.txt -u https://target.com/FUZZ

Port 445 (SMB):
  nmap -p 445 --script smb-vuln* <host>
  enum4linux -a <host>
  smbclient -L \\\\<host>\\ -N
  crackmapexec smb <host> --shares

Port 1433 (MSSQL):
  nmap -p 1433 --script ms-sql-info,ms-sql-empty-password <host>

Port 3306 (MySQL):
  nmap -p 3306 --script mysql-info,mysql-empty-password <host>

Port 3389 (RDP):
  nmap -p 3389 --script rdp-enum-encryption <host>
  # Check: NLA required or not, encryption level

Port 5985/5986 (WinRM):
  evil-winrm -i <host> -u <user> -p <password>
  # If port is open: test credential reuse

Version history analysis:
  # Cross-reference found versions against:
  searchsploit <service> <version>
  https://nvd.nist.gov/vuln/search
  https://www.exploit-db.com
```

***

## 4. Host Enumeration

Deep inspection of individual hosts, both from the outside before exploitation and from the inside after gaining access.

```
External host fingerprinting:
  # OS detection
  nmap -O <host>
  nmap -A <host>  # OS + version + scripts + traceroute

  # Banner grabbing
  nc -nv <host> <port>
  curl -I https://<host>

  # SNMP enumeration (often left open internally)
  onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <host>
  snmpwalk -v2c -c public <host>
  # Returns: running processes, installed software, user accounts, network interfaces

  # LDAP enumeration (domain joined hosts)
  ldapsearch -x -H ldap://<host> -b "DC=domain,DC=local"
  enum4linux-ng -A <host>

Internal host enumeration (post-foothold):
Windows:
  systeminfo                          # OS, patches, hardware
  whoami /all                         # current user, SID, privileges
  net user                            # local users
  net localgroup administrators       # local admins
  net share                           # shared resources
  netstat -ano                        # active connections and listening ports
  tasklist /svc                       # running processes with services
  wmic product get name,version       # installed software
  wmic service get Name,PathName,StartMode,State  # services
  reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run  # persistence

Linux:
  id && whoami                        # current user context
  cat /etc/passwd                     # user accounts
  cat /etc/hosts                      # internal hostname resolution
  ss -tulpn                           # listening ports and services
  ps aux                              # running processes
  find / -perm -4000 2>/dev/null      # SUID binaries
  find / -writable -type f 2>/dev/null # writable files
  cat ~/.bash_history                 # command history
  env                                 # environment variables (may contain creds)
  crontab -l && cat /etc/crontab      # scheduled tasks
```

***

## Pillaging - Integrated Across All Phases

Pillaging is not a separate standalone stage. It is the sensitive data collection component of both post-exploitation and information gathering performed locally on a compromised host.

```
What to look for on compromised hosts:

Credentials:
  Windows:
  - SAM / NTDS.dit (password hashes)
  - Registry: HKLM\SYSTEM + HKLM\SAM
  - Browser saved passwords (Chrome, Firefox, Edge)
  - mRemoteNG / RDCMan config files (encrypted creds)
  - Unattend.xml (deployment files with base64 passwords)
  - web.config / application config files
  - PowerShell history: %APPDATA%\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
  - Sticky notes: %LOCALAPPDATA%\Packages\...\LocalState\plum.sqlite

  Linux:
  - /etc/shadow (hashed passwords)
  - ~/.ssh/id_rsa (private SSH keys)
  - ~/.ssh/known_hosts (internal hostnames)
  - ~/.bash_history (commands, inline passwords)
  - /var/www/html/ config files (DB connection strings)
  - /home/*/.*rc files (environment variables)

Network intelligence:
  - ARP cache (recently communicated hosts)
  - Routing tables (network segments)
  - /etc/hosts (internal DNS overrides)
  - Firewall rules (what is allowed from this host)

Sensitive data:
  - Customer databases
  - Financial spreadsheets
  - HR files
  - Source code repositories
  - Internal documentation (network diagrams, runbooks)
  Document location and existence only - do not exfiltrate unnecessarily
```

***

## Information Gathering Flow Summary

```
OSINT (passive, no target interaction)
    |
    v
Infrastructure Enumeration (DNS, CIDR, cloud, security controls)
    |
    v
Service Enumeration (port scan -> banner grab -> version check -> CVE lookup)
    |
    v
Host Enumeration (OS fingerprint -> config -> role in network)
    |
    v
Exploitation (using all gathered intelligence)
    |
    v
Post-Exploitation + Pillaging (internal information gathering on compromised host)
    |
    v
Feed findings back into Infrastructure/Service/Host Enumeration
for lateral movement targets
    |
    v
Repeat until all in-scope systems assessed or objective reached
```
