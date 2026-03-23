# Initial Enumeration of the Domain

Starting an internal AD assessment from an unauthenticated position with only a network range is one of the most realistic and common scenarios you will encounter. The client wants to understand what an attacker can achieve if they gain physical or network access without any advance knowledge of the environment. Your job is to go from zero information to a valid domain user account as efficiently and quietly as the engagement style allows.

## Assessment Setup Options

The way a client delivers internal access varies significantly, and each setup has implications for what you can and cannot do. The most common arrangements are:

- A Linux pentest VM inside the client's infrastructure that calls back to a jump host over VPN, accessed via SSH
- A physical device plugged into an ethernet port that calls back over VPN
- A physical presence at the office with a laptop on an ethernet port
- A Linux VM in AWS or Azure with access to the internal network via public key SSH
- VPN access into the internal network (note: LLMNR and NBT-NS poisoning attacks generally do not work over VPN since you are not on the same broadcast domain)
- A managed Windows workstation, either in person or via VDI, with varying levels of tool access

For this engagement, Inlanefreight has provided a custom Linux pentest VM calling back to a jump host, a Windows host for tool loading, grey-box information (network range `172.16.5.0/23` only), and a standard domain user account (`htb-student`) for the Windows host. Testing is non-evasive, and we start unauthenticated. The goal is to enumerate the environment, identify critical hosts, find a path to domain user credentials, and document everything along the way.

## Phase 1 - Passive Host Identification

Before sending a single active packet, listening to what the network is already broadcasting gives free intelligence with no risk of detection.

**Wireshark** captures layer 2 traffic on the local segment. ARP requests and replies immediately reveal active hosts communicating on the network. MDNS traffic reveals hostnames. Starting Wireshark on the attack host:

```bash
sudo -E wireshark
```

From the initial capture, ARP traffic reveals `172.16.5.5`, `172.16.5.25`, `172.16.5.50`, `172.16.5.100`, and `172.16.5.125`. MDNS reveals the hostname `ACADEMY-EA-WEB01`. On headless hosts without a GUI, tcpdump achieves the same result and can save output to a `.pcap` file for later analysis in Wireshark:

```bash
sudo tcpdump -i ens224
```

**Responder in Analyze mode** extends passive capture to LLMNR, NBT-NS, and MDNS traffic without sending any poisoned packets. The `-A` flag puts it in passive listen mode only:

```bash
sudo responder -I ens224 -A
```

Responder in this mode surfaces additional hosts and domain information that ARP alone would miss. Any host that sends an LLMNR or NBT-NS query appears in the output with its name and IP, growing the target list without any active probing.

## Phase 2 - Active Host Discovery

With a baseline from passive captures, active checks confirm what is live and fill in gaps.

**fping** sweeps the entire `/23` range using ICMP in a round-robin fashion, which is faster than standard ping against multiple hosts because it does not wait for each host to respond before moving on. The `-a` flag shows alive hosts only, `-s` prints statistics, `-g` generates the target list from CIDR notation, and `-q` suppresses per-target noise:

```bash
fping -asgq 172.16.5.0/23
```

Output from the sweep:

```
172.16.5.5
172.16.5.25
172.16.5.50
172.16.5.100
172.16.5.125
172.16.5.200
172.16.5.225
172.16.5.238
172.16.5.240

     510 targets
       9 alive
```

Nine live hosts from 510 addresses. These go into a `hosts.txt` file for Nmap.

## Phase 3 - Service Enumeration with Nmap

The fping output tells us what is alive. Nmap tells us what those hosts are and what they are running:

```bash
sudo nmap -v -A -iL hosts.txt -oN /home/htb-student/Documents/host-enum
```

The `-A` flag enables OS detection, version detection, script scanning, and traceroute. Always use `-oA` or `-oN` to save output. Losing scan results and having to rescan is wasted time and adds unnecessary noise to the network.

The DC discovery result is immediately recognizable from its port profile. A host running DNS (53), Kerberos (88), RPC (135), NetBIOS (139), LDAP (389/636), SMB (445), and Global Catalog (3268/3269) is a Domain Controller. The SSL certificate on LDAP confirms the identity:

```
PORT     STATE SERVICE
53/tcp   open  domain
88/tcp   open  kerberos-sec
389/tcp  open  ldap          (Domain: INLANEFREIGHT.LOCAL, Site: Default-First-Site-Name)
636/tcp  open  ssl/ldap
| ssl-cert: Subject Alternative Name: DNS:ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
3268/tcp open  ldap
3269/tcp open  ssl/ldap
```

`ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL` at `172.16.5.5` is the primary Domain Controller. This is now the most important IP in the list -- nearly every subsequent attack and enumeration step will target or reference it.

The scan of `172.16.5.100` surfaces something worth flagging immediately:

```
PORT     STATE SERVICE
80/tcp   open  http          Microsoft IIS httpd 7.5
445/tcp  open  microsoft-ds  Windows Server 2008 R2 Standard 7600
1433/tcp open  ms-sql-s      Microsoft SQL Server 2008 R2 10.50.1600.00 RTM
```

Windows Server 2008 R2 is end-of-life. SQL Server 2008 R2 RTM means no service packs have been applied. This is a legacy host running unpatched software -- a strong candidate for EternalBlue, MS08-067, or other older exploits. Before running any exploits against it, the client should be informed and provide written approval, since attacking it risks destabilizing the service.

Note the SMB signing status in the results: `Message signing enabled but not required`. Hosts where SMB signing is not enforced are candidates for SMB relay attacks later in the assessment.

## Phase 4 - Username Enumeration with Kerbrute

With the DC identified at `172.16.5.5`, the next objective before attempting any credential attacks is building a valid username list. Kerbrute does this by exploiting a Kerberos behavior: the KDC returns a different error code for a valid username with a wrong password versus a completely unknown username. This means Kerbrute can confirm whether an account exists without submitting an actual password, and username-only enumeration does not increment the failed login counter and will not trigger lockouts.

Build from source to ensure you know what you are running:

```bash
sudo git clone https://github.com/ropnop/kerbrute.git
cd kerbrute
sudo make all
sudo mv dist/kerbrute_linux_amd64 /usr/local/bin/kerbrute
```

Run against the DC using the `jsmith.txt` list from the statistically-likely-usernames repository:

```bash
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users
```

Result:

```
[+] VALID USERNAME: jjones@INLANEFREIGHT.LOCAL
[+] VALID USERNAME: sbrown@INLANEFREIGHT.LOCAL
[+] VALID USERNAME: tjohnson@INLANEFREIGHT.LOCAL
...
Done! Tested 48705 usernames (56 valid) in 9.940 seconds
```

56 confirmed valid domain accounts in under 10 seconds. The `-o` flag saves the output to a file, which feeds directly into the next phase: password spraying.

## SYSTEM Level Access as an Alternative Path

If a valid domain user account cannot be obtained through enumeration and spraying, gaining SYSTEM on any domain-joined host is an equally valid path forward. A SYSTEM-level shell on a domain-joined machine can enumerate AD by impersonating the computer account, which is just another type of principal in the directory. Common routes to SYSTEM include:

- Remote exploits against unpatched services (EternalBlue against the identified Windows 2008 host, for example)
- Abusing services running as SYSTEM or exploiting SeImpersonatePrivilege via Juicy Potato on older Windows versions
- Local privilege escalation vulnerabilities on accessible Windows hosts
- Gaining local admin through weak credentials and using Psexec to spawn a SYSTEM shell

Once SYSTEM is obtained, BloodHound, PowerView, Inveigh, Kerberoasting, and token impersonation all become available options.

## A Word on Scan Caution

The `-A` flag in Nmap runs scripted checks that include active probing. Some NSE scripts perform actions that could destabilize fragile hosts, particularly industrial control systems, HVAC controllers, or legacy embedded devices that may exist in the `172.16.5.0/23` range. Always understand what a scan does before running it against a client's production network. If the engagement is non-evasive but the client has flagged specific hosts as sensitive, those should be approached carefully or excluded from aggressive scans entirely.

All scan output should be saved with `-oA` to produce XML, grepable, and normal format outputs simultaneously. The XML output can be imported directly into tools like Metasploit or parsed programmatically, and having all formats from the start means you will not need to rescan later just to feed output into a different tool.
