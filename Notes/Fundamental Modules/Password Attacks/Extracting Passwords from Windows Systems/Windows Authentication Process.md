# Windows Authentication Process

Understanding the Windows authentication architecture is a prerequisite for credential-based attacks during a penetration test. Knowing which component holds credentials, in what format, and under what access conditions determines which attack techniques are applicable in a given scenario.

<img width="2480" height="1424" alt="image" src="https://github.com/user-attachments/assets/76937bdc-0927-4a67-8e5f-17329b24bf54" />

## Authentication Flow Overview

When a user logs into a Windows system interactively, the process involves a chain of coordinated components:

1. **WinLogon** (`winlogon.exe`) receives the keypress interrupt from `Win32k.sys` via RPC and launches **LogonUI** to present the credential prompt
2. **Credential providers** (COM DLLs) collect the username and password from the user and pass them to WinLogon
3. **WinLogon** forwards the credentials to **LSASS** (`lsass.exe`) for authentication
4. **LSASS** selects the appropriate authentication package (via the `Negotiate` function in `lsasrv.dll`) and either checks credentials against the local **SAM** database or forwards the request to a **Domain Controller** for validation against **NTDS.dit**
5. On success, LSASS issues a security token containing the user's SID and group memberships, which becomes the basis for all subsequent access control decisions in that session

## LSASS

The [Local Security Authority Subsystem Service](https://en.wikipedia.org/wiki/Local_Security_Authority_Subsystem_Service) (`%SystemRoot%\System32\lsass.exe`) is the single most important process from an attacker's perspective. It enforces security policy, handles all authentication, and — critically — retains credential material in memory for the duration of active sessions. LSASS is mapped to [MITRE ATT&CK T1003.001](https://attack.mitre.org/techniques/T1003/001/) (OS Credential Dumping: LSASS Memory) and is one of the most consistently observed targets across real-world intrusions.

The authentication DLLs loaded by LSASS and their roles are:

| DLL | Role |
|---|---|
| `lsasrv.dll` | LSA Server service; enforces security policy and hosts the `Negotiate` function that selects between NTLM and Kerberos |
| `msv1_0.dll` | Handles local machine logons and NTLM authentication |
| `kerberos.dll` | Handles Kerberos-based authentication for domain-joined systems |
| `samsrv.dll` | Interfaces with the SAM database for local account management |
| `netlogon.dll` | Handles network-based logon operations including pass-through authentication to domain controllers |
| `ntdsa.dll` | Directory System Agent; manages the NTDS.dit database and LDAP queries — loaded only on Domain Controllers |

### What LSASS Holds in Memory

After a successful logon, LSASS retains credential material in its process memory for the lifetime of the session:

- **NTLM hashes** — for all users with active sessions on the machine
- **Kerberos TGTs and service tickets** — for domain-authenticated sessions
- **Plaintext passwords** — on systems where WDigest authentication is enabled (default off since Windows 8.1/Server 2012R2, but re-enableable via registry)
- **Cached domain credentials** — a small number of recently authenticated domain accounts are cached locally (DCC2 hashes) to allow logon when the domain controller is unreachable

Because of this, LSASS memory is the primary target for tools like Mimikatz (`sekurlsa::logonpasswords`), Task Manager process dumps, and ProcDump during post-exploitation.

## SAM Database

The [Security Account Manager](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc756748(v=ws.10)?redirectedfrom=MSDN) (SAM) database stores credentials for all **local** user accounts on a Windows system. It is located at:

```
%SystemRoot%\system32\config\SAM
```

and is mounted in the registry at `HKLM\SAM`. The file is locked by LSASS at all times during system operation and cannot be accessed or copied directly while Windows is running. Accessing it requires SYSTEM-level privileges.

Passwords are stored as **NTLM hashes** (and historically LM hashes, now disabled by default). The SAM is not used in domain environments — when a system is domain-joined, domain account authentication is delegated to the DC, which validates against NTDS.dit instead.

### SYSKEY

Introduced in Windows NT 4.0, SYSKEY (`syskey.exe`) encrypts the SAM database on disk using a system-generated 128-bit key, stored in the `HKLM\SYSTEM` registry hive as the **boot key** (also called the system key or SysKey). Extracting SAM hashes offline therefore requires both the SAM file and the SYSTEM hive — the SYSTEM hive is needed to derive the boot key used to decrypt the SAM. This is why tools like Impacket's `secretsdump.py` and `reg save` commands always target both:

```
reg save HKLM\SAM sam.bak
reg save HKLM\SYSTEM system.bak
```

## Credential Manager

Credential Manager is a built-in Windows vault that stores credentials used to access network resources, websites, and applications. Entries are stored per-user profile and encrypted at:

```
C:\Users\[Username]\AppData\Local\Microsoft\Vault\
C:\Users\[Username]\AppData\Local\Microsoft\Credentials\
```

The vault holds two types of credentials:

- **Windows Credentials** — NTLM or Kerberos credentials for network resources, mapped drives, and scheduled tasks
- **Web Credentials** — credentials saved by Internet Explorer or legacy Edge for websites

Credentials in the vault are protected using the [Data Protection API](https://learn.microsoft.com/en-us/windows/win32/api/dpapi/) (DPAPI) with the user's master key. Because DPAPI is tied to the user's logon credentials, an attacker with access to the user's session (or their NTLM hash and the DPAPI master key) can decrypt vault entries. Tools such as Mimikatz (`dpapi::cred`), [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI), and Impacket's `dpapi.py` are commonly used for this purpose during assessments.

## NTDS.dit

In domain environments, the authoritative credential store is `NTDS.dit`, an Extensible Storage Engine (ESE) database maintained by the [Directory System Agent](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc961760(v=technet.10)) on every Domain Controller. Its default location is:

```
%SystemRoot%\NTDS\ntds.dit
```

NTDS.dit stores the complete Active Directory dataset, including:

- All **user account objects** with their NTLM password hashes (and LM hashes where present)
- **Machine account** hashes for every domain-joined computer
- **Group memberships**, Group Policy Objects, and trust relationships
- **Password history** — typically the last 24 hashes per user, depending on policy

Like the SAM, NTDS.dit is locked by the NTDS service while Active Directory is running and cannot be directly copied. Attackers bypass this using Volume Shadow Copies (`vssadmin create shadow /for=C:`), the `ntdsutil` utility, or API-level approaches such as the Directory Replication Service (DRS) API used by `secretsdump.py`'s DCSync attack — which impersonates a Domain Controller replication request to pull hashes remotely without touching the filesystem at all.

Decrypting the extracted hashes from NTDS.dit requires the boot key from the DC's `HKLM\SYSTEM` hive, just as with the SAM.

## Credential Storage Reference

| Store | Location | What It Holds | Access Required |
|---|---|---|---|
| **SAM** | `%SystemRoot%\system32\config\SAM` | Local account NTLM hashes | SYSTEM |
| **LSASS memory** | `lsass.exe` process | NTLM hashes, Kerberos tickets, cached creds, plaintext (WDigest) | SYSTEM / SeDebugPrivilege |
| **Credential Manager** | `AppData\Local\Microsoft\Vault` | DPAPI-encrypted network/web credentials | User session or DPAPI master key |
| **NTDS.dit** | `%SystemRoot%\NTDS\ntds.dit` (DC only) | All domain account hashes, machine hashes, history | Domain Admin / DCSync rights |
| **Registry (cached)** | `HKLM\SECURITY\Cache` | DCC2 hashes for last N domain logons | SYSTEM |
