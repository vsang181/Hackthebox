# Credential Hunting in Windows

Credential hunting is the practice of systematically searching a compromised system for stored credentials across the file system, applications, and configuration files. On an IT admin's workstation the attack surface is particularly rich — admins routinely store credentials in scripts, configuration files, remote management tools, and browsers, often in cleartext or weakly protected formats.

## Key Terms to Search For

Effective credential hunting starts with knowing what to search for rather than guessing blindly. High-value search terms include:

- `password`, `passwd`, `pwd`, `passphrase`, `passkey`
- `username`, `user account`, `credentials`, `creds`, `login`
- `dbpassword`, `dbcredential`, `connectionstring`
- `configuration`, `secret`, `key`, `token`

Tailor the terms to the system's function. On an IT admin workstation, terms like `admin`, `ssh`, `vnc`, `rdp`, and `winrm` alongside the above are particularly productive.

## Tool 1: Windows Search (GUI)

When a graphical session is available, the built-in Windows Search bar provides the fastest initial sweep. By default it indexes OS settings, installed applications, and the file system. Entering terms like `password` or `credentials` can surface documents, spreadsheets, scripts, and config files that explicitly mention those terms without requiring any additional tooling.

## Tool 2: LaZagne

[LaZagne](https://github.com/AlessandroZ/LaZagne) is an open-source credential recovery tool that targets the insecure storage mechanisms of dozens of applications simultaneously. It is catalogued under [MITRE ATT&CK T1555](https://attack.mitre.org/software/S0349/) and supports over 35 browsers on Windows alone alongside system tools, mail clients, chat applications, and more.

Key modules and what they target:

| Module | Targets |
|---|---|
| `browsers` | Chrome, Firefox, Edge, Opera, and 30+ others |
| `chats` | Skype and other messaging applications |
| `mails` | Outlook, Thunderbird |
| `memory` | KeePass databases, LSASS process memory |
| `sysadmin` | WinSCP, OpenVPN, PuTTY, FileZilla config files |
| `windows` | LSA secrets, Credential Manager vault entries |
| `wifi` | Saved wireless network pre-shared keys |

Transfer the [standalone LaZagne.exe](https://github.com/AlessandroZ/LaZagne/releases/) to the target — over RDP this can be done via copy-paste directly into the session if using `xfreerdp`. Then run all modules at once:

```cmd
C:\Users\bob\Desktop> start LaZagne.exe all
```

To see exactly what LaZagne is doing in the background as it runs, add the `-vv` flag for verbose output. To save results quietly to a file without printing to screen:

```cmd
LaZagne.exe all -quiet -oA
```

Example output when credentials are found:

```
########## User: bob ##########

------------------- Winscp passwords -----------------

[+] Password found !!!
URL: 10.129.202.51
Login: admin
Password: SteveisReallyCool123
Port: 22
```

WinSCP is a common credential source on IT admin workstations since admins frequently save SSH and SFTP session credentials for convenience. Similarly, browsers are high-value targets — Chrome, Edge, and Firefox all encrypt their stored credentials, but tools like [firefox_decrypt](https://github.com/unode/firefox_decrypt) and [decrypt-chrome-passwords](https://github.com/ohyicong/decrypt-chrome-passwords) (and LaZagne itself) can recover them from the current user's session context.

> **Detection note:** LaZagne's execution produces distinctive command-line patterns and accesses known credential database file paths across the system. EDR platforms and Sigma rules specifically detect keywords like `browser` combined with LaZagne's known execution patterns. Running it with `-quiet -oA` reduces screen output but does not reduce file system access telemetry.

## Tool 3: findstr

[findstr](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/findstr) is a built-in Windows utility similar to Linux's `grep`. It can recursively search file contents across multiple file types simultaneously, making it useful for locating credentials embedded in configuration and script files:

```cmd
# Search recursively across common config and script file types
C:\> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml

# Search specifically in a user's home directory
C:\> findstr /S /I /C:"password" "C:\Users\*" *.txt *.ini *.cfg *.config *.xml

# Search for cpassword values in SYSVOL (Group Policy Preferences)
C:\> findstr /S /I cpassword \\<DOMAIN>\sysvol\<DOMAIN>\policies\*.xml
```

The flags used are:

| Flag | Meaning |
|---|---|
| `/S` | Search current directory and all subdirectories recursively |
| `/I` | Case-insensitive matching |
| `/M` | Print only the filename, not every matching line |
| `/C:"term"` | Treat the argument as a literal string rather than a pattern |

## High-Value Locations to Check Manually

Beyond automated tools, certain locations on Windows environments are reliably worth checking:

- **SYSVOL scripts and Group Policy Preferences** — Group Policy Preference (GPP) XML files in `\\<DOMAIN>\SYSVOL` may contain `cpassword` fields, which are AES-256 encrypted passwords whose key was [publicly disclosed by Microsoft](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-gppref/2c15cbf0-f086-4c74-8b70-1f2fa45dd4be). The `gpp-decrypt` tool decrypts them instantly.
- **Unattend.xml and Sysprep files** — Windows deployment answer files frequently contain local administrator credentials in base64 or plaintext. Common paths to check:

```
C:\unattend.xml
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattend\Unattend.xml
C:\Windows\System32\Sysprep\unattend.xml
C:\Windows\System32\Sysprep\sysprep.xml
```

- **Web.config files** — On dev machines and IIS servers, `web.config` files commonly contain database connection strings with embedded credentials.
- **IT and dev shares** — Scripts, runbooks, and automation files on network shares often contain hardcoded credentials that were never cleaned up.
- **AD user and computer description fields** — Admins sometimes store passwords in the Description field of AD user or computer objects, visible to any authenticated domain user.
- **KeePass databases (.kdbx)** — If a KeePass database can be located on the file system or a share, the master password may be crackable with [hashcat mode 13400](https://hashcat.net/wiki/doku.php?id=example_hashes) or may be stored nearby.
- **Files named explicitly** — Always search for files named `pass.txt`, `passwords.docx`, `passwords.xlsx`, `creds.txt` on user systems, file shares, and [SharePoint](https://www.microsoft.com/en-us/microsoft-365/sharepoint/collaboration) sites.

## Credential Hunting Reference

| Method | Tool | Best For |
|---|---|---|
| Full application credential sweep | `LaZagne.exe all` | Browsers, WinSCP, mail clients, Wi-Fi |
| File system keyword search | `findstr /SIM /C:"password" *.xml *.cfg *.ps1` | Config files, scripts, deployment files |
| GPP password search | `findstr /S /I cpassword \\domain\sysvol\*.xml` | Group Policy Preference credentials |
| Browser-specific extraction | firefox_decrypt, decrypt-chrome-passwords | Targeted browser credential recovery |
| GUI search | Windows Search bar | Quick initial sweep with GUI access |
| Manual file check | Direct navigation | Unattend.xml, web.config, KeePass .kdbx |
