# Credential Hunting in Network Shares

Network shares in corporate environments are among the most consistently productive targets for credential hunting. Employees routinely store configuration files, deployment scripts, backup files, and internal documentation on shared drives — often containing credentials that were never intended to be accessible beyond a small team, but are readable by any authenticated domain user.

## Search Strategy Before Tooling

Before running automated scanners, a brief manual assessment saves significant time. Consider:

- **Which shares are worth targeting.** IT, DevOps, and Infrastructure shares are far higher value than Marketing or HR shares for credential material. SYSVOL and NETLOGON are always worth inspecting.
- **What keywords are relevant to the target environment.** For an English-speaking company, `passw`, `cred`, `secret`, `token`, and `key` are standard. For a German company, add `Benutzer`, `Kennwort`, and `Passwort`.
- **What file types are likely to contain credentials.** Prioritise `.ini`, `.cfg`, `.env`, `.xml`, `.ps1`, `.bat`, `.sh`, `.conf`, and office documents like `.xlsx` and `.docx`.
- **File naming patterns.** Files named `config`, `credentials`, `initial`, `backup`, or `setup` are commonly overlooked but frequently contain sensitive data.

For quick manual searches from PowerShell before scaling up to dedicated tools:

```powershell
# Search file contents recursively across a share
Get-ChildItem -Recurse -Include *.txt,*.xml,*.ini,*.ps1 \\Server\IT | Select-String -Pattern "passw"

# Search filenames for interesting terms
Get-ChildItem -Recurse \\Server\IT | Where-Object { $_.Name -match "passw|cred|secret|config" }
```

## Hunting from Windows

### Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler) is a C# tool designed to run on a domain-joined machine. It queries Active Directory for all computers in the domain, enumerates accessible SMB shares on each, recursively scans file paths and contents against a built-in set of regex patterns targeting sensitive data, and colour-codes results by severity. It requires no configuration for a basic run:

```cmd
Snaffler.exe -s
```

Output colour codes indicate priority:

| Colour | Meaning |
|---|---|
| Red | High confidence credential or sensitive data match |
| Yellow | Interesting file type or pattern worth reviewing |
| Green | Readable share or directory identified |
| Black | Share exists but is not accessible |

Two particularly useful flags for refining results:

- `-u` retrieves the user list from Active Directory and searches for references to those usernames inside files — useful for finding files that mention specific accounts
- `-i <sharename>` and `-n <sharename>` control which shares are included or excluded from the scan

From the example output, Snaffler flagged `unattend.xml` at `\\DC01.inlanefreight.local\ADMIN$\Panther\unattend.xml` as a Red hit matching the `KeepPassOrKeyInCode` rule — this is the Windows deployment answer file discussed in the Credential Hunting in Windows section, which frequently contains a plaintext or Base64-encoded `AdministratorPassword` field.

### PowerHuntShares

[PowerHuntShares](https://github.com/NetSPI/PowerHuntShares) is a PowerShell script that takes a different angle — rather than focusing on file content, it maps the entire share permission landscape of the domain and generates a full HTML report. This is particularly useful for identifying shares with excessive permissions (write or full control for non-admin users) that represent both a data exposure risk and a potential persistence or lateral movement path:

```powershell
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
```

The HTML report it generates is browsable and filterable, making manual review of large environments significantly more manageable than parsing raw text output. It enumerates share permissions, identifies high-risk shares, maps common share owners, and generates last-written and last-accessed timelines.

## Hunting from Linux

### MANSPIDER

[MANSPIDER](https://github.com/blacklanternsecurity/MANSPIDER) is the Linux-native equivalent of Snaffler for SMB share content searching. It supports regex-based filename and content matching, handles multiple file formats (DOCX, XLSX, PPTX, and plain text), downloads matching files to a local loot directory, and falls back to guest or null sessions when credentials fail. Running it via the official Docker container avoids dependency issues:

```bash
# Search file contents for "passw"
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider \
    10.129.234.121 -c 'passw' -u 'mendres' -p 'Inlanefreight2025!'

# Search filenames for credential-related terms
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider \
    10.129.234.121 -f 'passw|cred|secret|config' -u 'mendres' -p 'Inlanefreight2025!'

# Target only the IT share and search for SSH private keys by content
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider \
    10.129.234.121 --sharenames IT -c 'BEGIN .{1,10} PRIVATE KEY' -u 'mendres' -p 'Inlanefreight2025!'
```

All files matching the search criteria are downloaded to `./manspider/loot/` on the host for offline inspection.

Key MANSPIDER flags:

| Flag | Purpose |
|---|---|
| `-c <regex>` | Search file contents using regex |
| `-f <regex>` | Filter by filename using regex |
| `-e <ext>` | Only process files with specific extensions |
| `--sharenames <name>` | Restrict search to specific share names |
| `-t <int>` | Number of concurrent threads (default 5) |
| `-s <size>` | Skip files larger than this size (default 10MB) |
| `-H <hash>` | Authenticate using an NT hash instead of a password (Pass-the-Hash) |
| `-n` | List matches without downloading files |

### NetExec Spider

NetExec's `--spider` option provides a quick targeted search of a specific share without the overhead of a full MANSPIDER deployment:

```bash
# Search a specific share for files containing "passw"
nxc smb 10.129.234.121 -u mendres -p 'Inlanefreight2025!' \
    --spider IT --content --pattern "passw"

# Search for interesting filenames across all shares
nxc smb 10.129.234.121 -u mendres -p 'Inlanefreight2025!' \
    --spider C$ --pattern "config|cred|pass" --regex
```

The `--spider` module and its options are documented fully on the [NetExec wiki](https://www.netexec.wiki/smb-protocol/spidering-shares). It is the fastest option for a targeted sweep of a single known share but less comprehensive than MANSPIDER for full domain-wide enumeration.

## Tool Selection Reference

| Scenario | Tool | Platform |
|---|---|---|
| Domain-joined machine, full domain sweep | Snaffler | Windows |
| Share permission mapping with HTML report | PowerHuntShares | Windows (no domain join required) |
| Linux attack host, comprehensive content search | MANSPIDER (Docker) | Linux |
| Quick targeted search of a single share | NetExec `--spider` | Linux |
| Manual keyword search before tooling | `Get-ChildItem` + `Select-String` | Windows PowerShell |
| Manual recursive content search | `grep -r "passw" /mnt/share/` | Linux (mounted share) |
