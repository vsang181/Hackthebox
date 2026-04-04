## Attacking Applications Connecting to Services

Applications that connect to backend services like MSSQL, MySQL, or APIs frequently embed connection strings directly in their binaries. These strings contain credentials that can be extracted through static or dynamic analysis and reused to access backend services, pivot laterally, or attempt password spraying against other systems.  The two most common scenarios are ELF binaries on Linux and .NET DLL assemblies on Windows. 
***

## Path 1: ELF Binary Analysis with GDB and PEDA

[PEDA](https://github.com/longld/peda) (Python Exploit Development Assistance for GDB) extends standard GDB with enhanced register display, memory inspection, and visual output during debugging.  The goal is to intercept the connection string in memory at the exact moment it is passed to the database connection function. 

### Step 1: Load the Binary and Disassemble

```bash
gdb ./octopus_checker
```

Set Intel syntax for more readable output, then disassemble `main`:

```
gdb-peda$ set disassembly-flavor intel
gdb-peda$ disas main
```

Scan the disassembly for calls to database-related functions. The key target here is `SQLDriverConnect`, which receives the full ODBC connection string as an argument just before the connection attempt is made:

```asm
0x0000555555555607 <+433>:   call   0x5555555551b0 <SQLDriverConnect@plt>
```

### Step 2: Set a Breakpoint at SQLDriverConnect

Set the breakpoint at the PLT stub address for `SQLDriverConnect`, not at the call site. This catches the function exactly at the moment it is invoked, with all arguments already loaded into registers: 

```
gdb-peda$ b *0x5555555551b0
```

### Step 3: Run and Inspect the RDX Register

```
gdb-peda$ run
```

When the breakpoint is hit, PEDA dumps the current register state. The ODBC connection string is passed as the second argument in the `RDX` register on x86-64 Linux (System V ABI):

```
RDX: 0x7fffffffda70 ("DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=username;PWD=password;")
```

The full plaintext connection string is now visible, including the username and password.

### Key GDB Commands Reference

| Command | Purpose |
|---|---|
| `disas main` | Disassemble the main function |
| `b *0xADDRESS` | Set breakpoint at a specific memory address |
| `run` | Start execution |
| `info registers` | Print all current register values |
| `x/s $rdx` | Print the string at the address held in RDX |
| `x/s 0xADDRESS` | Print a string at a literal address |
| `c` | Continue execution after a breakpoint |

Once a connection string is recovered, test the credentials against:

- The local database service directly via `sqlcmd`, `mssqlclient.py`, or similar
- Other services on the same network via password spraying in case credentials are reused
- SMB, WinRM, or SSH if the service account corresponds to a domain or local account

***

## Path 2: .NET DLL Analysis with dnSpy

[dnSpy](https://github.com/dnSpyEx/dnSpy) decompiles .NET assemblies (`.dll` and `.exe`) directly back to readable C# or VB.NET source code without requiring the original project files.  It also functions as a full debugger, allowing runtime inspection of values. 

### Step 1: Identify the Assembly Type

Use PowerShell metadata inspection or simply open the file in dnSpy and look for the `.NETFramework` string in the metadata header to confirm it is a managed assembly:

```powershell
Get-FileMetaData .\MultimasterAPI.dll
```

```
.NETFramework,Version=v4.6.1
TFrameworkDisplayName.NET Framework 4.6.1
api/getColleagues
http://localhost:8081
```

The metadata already reveals a local API endpoint, giving context before even opening the decompiler.

### Step 2: Open in dnSpy and Navigate the Class Tree

Drag the DLL onto dnSpy. The left panel shows the assembly tree. Navigate to:

```
MultimasterAPI.dll -> MultimasterAPI -> Controllers -> ColleagueController
```

The decompiled C# source is displayed directly. Look for:

- `SqlConnection` or `MySqlConnection` objects
- `ConnectionString` property assignments
- `ConfigurationManager.ConnectionStrings` calls
- Hardcoded `Server=`, `UID=`, `Password=`, `PWD=` strings
  
Connection strings in .NET controllers are frequently found in a pattern like:

```csharp
SqlConnection conn = new SqlConnection(
    "Server=localhost;Database=Multimaster;User Id=api_user;Password=D3veL0pM3nT!;"
);
```

### Step 3: Search Across the Entire Assembly

Use dnSpy's Edit > Search Assemblies feature and search for terms like:

- `password`
- `connectionstring`
- `PWD=`
- `UID=`
- `Server=`

This surfaces matches across all classes in the DLL without manually navigating every namespace. 

***

## Comparison of Techniques

| Aspect | ELF Binary (GDB/PEDA) | .NET DLL (dnSpy) |
|---|---|---|
| Platform | Linux | Windows |
| Analysis type | Dynamic (runtime breakpoint) | Static (decompilation) |
| Credential location | Register at time of DB call | Source code / class properties |
| Obfuscation impact | Low - credentials are in memory regardless | High - de4dot may be needed first |
| Tools | GDB, PEDA | dnSpy, de4dot |

***

## Post-Exploitation Use of Recovered Credentials

Credentials recovered from either approach should be tested against all available services in the environment, not just the intended database.  Common reuse targets include: 

- MSSQL via `mssqlclient.py -windows-auth` or `sqlcmd`
- SMB shares via `smbclient` or `crackmapexec`
- WinRM via `evil-winrm`
- SSH if the binary runs on a Linux server in a mixed environment
- Active Directory via password spraying with `kerbrute` if the credential follows a pattern suggesting a domain account
