# Interacting with Common Services

Understanding how to interact with common services is a foundational skill before attempting to attack them. You need to know what a service does, how legitimate users connect to it, and what tools are available -- only then can you identify what is misconfigured or exploitable.

## File Share Services

File sharing services transfer files between systems and users. Internal environments typically rely on SMB, NFS, FTP, TFTP, and SFTP, while cloud adoption has added services like Dropbox, Google Drive, OneDrive, AWS S3, and Azure Blob Storage. A pentest may encounter both, and the interaction methods differ between them.

### SMB -- Windows

SMB is the dominant file sharing protocol in Windows environments. The simplest interaction is through the GUI via the Run dialog (`[WINKEY] + [R]`), where entering a UNC path like `\\192.168.220.129\Finance\` will open the share directly if access is permitted or prompt for credentials if not.

From the command line, [dir](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/dir) lists share contents and [net use](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/gg651155(v=ws.11)) maps a share to a drive letter, enabling you to interact with it as if it were local storage:

```cmd
# Map the share anonymously
net use n: \\192.168.220.129\Finance

# Map with explicit credentials
net use n: \\192.168.220.129\Finance /user:plaintext Password123

# Count all files recursively
dir n: /a-d /s /b | find /c ":\"

# Search for files with credential-related names
dir n:\*cred* /s /b
dir n:\*secret* /s /b

# Search for the string "cred" inside all files
findstr /s /i cred n:\*.*
```

The `dir` flags used above break down as follows: `/a-d` lists files only (excluding directories), `/s` recurses into subdirectories, and `/b` outputs bare filenames with no header or summary. More examples are available in the [findstr documentation](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/findstr#examples).

PowerShell provides the same functionality through cmdlets. `Get-ChildItem` replaces `dir`, `New-PSDrive` replaces `net use`, and `Select-String` replaces `findstr`. When authenticating, PowerShell requires credentials to be wrapped in a [PSCredential object](https://docs.microsoft.com/en-us/dotnet/api/system.management.automation.pscredential) rather than passed as plain strings:

```powershell
# Build the credential object
$username = 'plaintext'
$password = 'Password123'
$secpassword = ConvertTo-SecureString $password -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential $username, $secpassword

# Map the drive with credentials
New-PSDrive -Name "N" -Root "\\192.168.220.129\Finance" -PSProvider "FileSystem" -Credential $cred

# Count all files
N:
(Get-ChildItem -File -Recurse | Measure-Object).Count

# Search by filename pattern
Get-ChildItem -Recurse -Path N:\ -Include *cred* -File

# Search by file content (equivalent to findstr)
Get-ChildItem -Recurse -Path N:\ | Select-String "cred" -List
```

### SMB -- Linux

Linux mounts SMB shares using the `cifs` kernel module. The `cifs-utils` package must be installed first (`sudo apt install cifs-utils`). Credentials can be passed inline or stored in a separate file to avoid exposing them in the process list:

```bash
# Mount with inline credentials
sudo mkdir /mnt/Finance
sudo mount -t cifs -o username=plaintext,password=Password123,domain=. //192.168.220.129/Finance /mnt/Finance

# Mount using a credential file (safer)
mount -t cifs //192.168.220.129/Finance /mnt/Finance -o credentials=/path/credentialfile
```

The credential file format is:

```
username=plaintext
password=Password123
domain=.
```

Once mounted, standard Linux tools work against the share as if it were a local directory:

```bash
# Find files by name
find /mnt/Finance/ -name *cred*

# Search file contents recursively
grep -rn /mnt/Finance/ -ie cred
```

## Email

Email requires two separate protocols -- one for sending and one for receiving. SMTP handles outbound delivery, while POP3 and IMAP handle inbound retrieval. [Evolution](https://wiki.gnome.org/Apps/Evolution) is a full-featured Linux mail client that supports both and integrates well with testing workflows:

```bash
sudo apt-get install evolution
# If a sandbox error occurs on launch:
export WEBKIT_FORCE_SANDBOX=0 && evolution
```

When configuring a connection, use SMTPS or IMAPS where possible. The "Check for Supported Types" option in Evolution's authentication settings confirms which methods the server supports before committing to a configuration.

## Databases

Relational databases are almost universally present in enterprise environments. The two most common targets are MSSQL and MySQL. Interaction methods fall into three categories: command-line utilities, programming language connectors, and GUI tools.

### Command-line utilities

For MSSQL from Linux, [sqsh](https://en.wikipedia.org/wiki/Sqsh) provides an interactive SQL shell with support for variables, piping, and scripting. From Windows, [sqlcmd](https://docs.microsoft.com/en-us/sql/tools/sqlcmd-utility) is the native Microsoft utility:

```bash
# Linux - MSSQL via sqsh
sqsh -S 10.129.20.13 -U username -P Password123

# Windows - MSSQL via sqlcmd
sqlcmd -S 10.129.20.13 -U username -P Password123
```

For MySQL, the `mysql` binary is available on both platforms:

```bash
# Linux
mysql -u username -pPassword123 -h 10.129.20.13

# Windows
mysql.exe -u username -pPassword123 -h 10.129.20.13
```

### GUI tools

[MySQL Workbench](https://dev.mysql.com/downloads/workbench/) is the official MySQL GUI, and [SQL Server Management Studio (SSMS)](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) covers MSSQL but is Windows-only. For cross-platform use, [dbeaver](https://github.com/dbeaver/dbeaver) supports MySQL, MSSQL, PostgreSQL, and many others from a single interface on Linux, macOS, and Windows:

```bash
# Install from .deb package
sudo dpkg -i dbeaver-<version>.deb

# Launch
dbeaver &
```

Once connected via any method, [Transact-SQL statements](https://docs.microsoft.com/en-us/sql/t-sql/statements/statements?view=sql-server-ver15) are used to enumerate databases, tables, and sensitive data. Attacks against MSSQL and MySQL -- including `xp_cmdshell` abuse -- are covered in later sections of this module.

## Tool Reference

| Service | Tools |
|---|---|
| SMB | [smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html), [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec), [SMBMap](https://github.com/ShawnDEvans/smbmap), [Impacket](https://github.com/SecureAuthCorp/impacket) (`psexec.py`, `smbexec.py`) |
| FTP | [ftp](https://linux.die.net/man/1/ftp), [lftp](https://lftp.yar.ru/), [ncftp](https://www.ncftp.com/), [filezilla](https://filezilla-project.org/), [crossftp](http://www.crossftp.com/) |
| Email | [Thunderbird](https://www.thunderbird.net/en-US/), [Evolution](https://wiki.gnome.org/Apps/Evolution), [Claws](https://www.claws-mail.org/), [mutt](http://www.mutt.org/), [swaks](http://www.jetmore.org/john/code/swaks/), [sendEmail](https://github.com/mogaal/sendemail) |
| Databases | [mssql-cli](https://github.com/dbcli/mssql-cli), [mycli](https://github.com/dbcli/mycli), [mssqlclient.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/mssqlclient.py), [dbeaver](https://github.com/dbeaver/dbeaver), [MySQL Workbench](https://dev.mysql.com/downloads/workbench/), [SSMS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms) |

## Troubleshooting Connection Failures

When a connection attempt fails, the cause almost always falls into one of these categories:

- **Authentication** -- wrong credentials, account locked, or an unsupported authentication method
- **Privileges** -- the account exists and authenticates but lacks the rights to access the target resource
- **Network connectivity** -- a firewall rule, routing issue, or VLAN separation is blocking the traffic
- **Protocol support** -- the client and server do not share a compatible protocol version (common with SMBv1 vs SMBv2/v3)
- **Missing packages** -- a required client utility or library is not installed on the attacking machine

Error codes returned by services are often specific enough to narrow down the cause quickly. Searching the exact error string in Microsoft's documentation or community forums almost always surfaces a known resolution.
