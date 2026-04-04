## Attacking Thick Client Applications

Thick client applications run locally on the user's machine rather than through a browser, performing significant processing on the client side. They are common in enterprise environments (ERP systems, CRM tools, internal service utilities) and are typically built with .NET, Java, C++, or older technologies.  Because they store data locally, embed logic in binaries the user can interact with directly, and often communicate with backend databases or application servers, they offer a distinct and often underestimated attack surface. 

***

## Architecture Types

| Type | Description | Security Implication |
|---|---|---|
| Two-tier | Client talks directly to the database | Database connection strings often embedded in the binary or config files |
| Three-tier | Client talks to an app server, which talks to the database | More secure by design, but HTTP/HTTPS traffic is interceptable |

***

## Common Vulnerabilities

- Hardcoded credentials and API keys in binaries or config files
- Sensitive data stored unencrypted in local files, the registry, or memory
- DLL hijacking through insecure library loading paths
- SQL injection via client-side constructed queries in two-tier apps
- Insecure or unencrypted network communication
- Weak or missing session management
- Improper error handling revealing internal stack traces or paths
- Buffer overflows in older C/C++ applications

***

## Penetration Testing Methodology

Thick client testing follows four main phases, each requiring a specific toolset:

### 1. Information Gathering

Identify the programming language, runtime, architecture, and what external services the application communicates with.

| Tool | Purpose |
|---|---|
| [CFF Explorer](https://ntcore.com/?page_id=388) | PE file header analysis, imports, exports |
| [Detect It Easy](https://github.com/horsicq/Detect-It-Easy) | Identify packers, protectors, compilers, .NET vs native |
| [Process Monitor](https://learn.microsoft.com/en-us/sysinternals/downloads/procmon) | Real-time file system, registry, and network activity |
| [Strings](https://learn.microsoft.com/en-us/sysinternals/downloads/strings) | Extract readable strings from binaries |

### 2. Client-Side Static and Dynamic Analysis

Reverse engineer the binary, read decompiled source code, and observe runtime behaviour in memory.

| Tool | Purpose |
|---|---|
| [dnSpy](https://github.com/dnSpyEx/dnSpy) | .NET decompiler and debugger, read and modify C# source |
| [de4dot](https://github.com/de4dot/de4dot) | .NET deobfuscator and unpacker |
| [x64dbg](https://x64dbg.com/) | Windows debugger for native x86/x64 binaries |
| [Ghidra](https://www.ghidra-sre.org/) | NSA-developed reverse engineering framework |
| [IDA](https://hex-rays.com/ida-pro/) | Industry-standard disassembler and decompiler |
| [JADX](https://github.com/skylot/jadx) | Java and Android APK decompiler |
| [Frida](https://frida.re/) | Dynamic instrumentation and runtime hooking |

### 3. Network Traffic Analysis

Capture and inspect traffic between the thick client and backend services. Three-tier applications communicating over HTTP can be proxied through Burp Suite exactly like a web application.

| Tool | Purpose |
|---|---|
| [Wireshark](https://www.wireshark.org/) | Full packet capture and protocol analysis |
| [TCPView](https://learn.microsoft.com/en-us/sysinternals/downloads/tcpview) | Live view of active TCP/UDP connections per process |
| [Burp Suite](https://portswigger.net/burp) | Intercept and modify HTTP/HTTPS from proxy-aware apps |
| [tcpdump](https://www.tcpdump.org/) | CLI packet capture for Linux environments |

### 4. Server-Side Testing

Once backend endpoints are identified from traffic analysis or decompilation, test them against the OWASP Top Ten as you would any web application.

***

## Hands-On: Extracting Hardcoded Credentials from a .NET Thick Client

This scenario walks through a realistic credential extraction chain starting from an SMB share discovery.

### Step 1: Discover the Executable via SMB

Exploring the NETLOGON share exposes `RestartOracle-Service.exe`. Running it from the command line produces no visible output, suggesting it runs something silently.

### Step 2: Monitor with ProcMon

Load ProcMon64, set a filter for the process name, and run the executable. Observation: a temp `.bat` file is created in `C:\Users\Matt\AppData\Local\Temp` and immediately deleted.

### Step 3: Prevent File Deletion to Capture the Batch File

Remove delete permissions from the Temp folder so the cleanup cannot complete:

Right-click `C:\Users\<user>\AppData\Local\Temp` > Properties > Security > Advanced > Select your user > Disable inheritance > Convert to explicit permissions > Edit > Show advanced permissions > uncheck "Delete subfolders and files" and "Delete" > Apply.

Run the executable again. The batch file (e.g., `6F39.bat`) is now captured.

### Step 4: Read the Batch Script

The batch script reveals it:
1. Checks the username against an allowlist (`matt`, `frankytech`, `ev4si0n`)
2. Writes a large base64-encoded blob to `c:\programdata\oracle.txt`
3. Creates a PowerShell script `monta.ps1` that decodes the blob into `restart-service.exe`
4. Executes the decoded binary
5. Deletes all three artefacts

### Step 5: Preserve the Dropped Files

Edit the batch script to remove all `del` lines, then run it. The files `oracle.txt`, `monta.ps1`, and `restart-service.exe` are now preserved.

### Step 6: Analyse restart-service.exe in x64dbg

Load the binary in [x64dbg](https://x64dbg.com/). Before starting, go to Options > Preferences and uncheck everything except Exit Breakpoint to avoid breaking on DLL load events. After attaching, right-click in the CPU view and select "Follow in Memory Map."

Look for a memory-mapped region with:

- Size: `0x3000`
- Type: `MAP`
- Protection: `-RW--`

Double-click to inspect it. The `MZ` magic bytes in the ASCII column confirm a DOS MZ executable is mapped there. Right-click the entry in the Memory Map pane and select "Dump Memory to File" to export it.

### Step 7: Confirm It Is a .NET Assembly

Run Strings on the dumped binary:

```powershell
.\strings64.exe .\restart-service_00000000001E0000.bin
```

```
.NETFramework,Version=v4.0,Profile=Client
FrameworkDisplayName
.NET Framework 4 Client Profile
```

The MZ executable embedded in memory is a .NET assembly.

### Step 8: Deobfuscate and Decompile

Run [de4dot](https://github.com/de4dot/de4dot) on the dump to strip obfuscation and restore readable symbol names: 

```bash
de4dot restart-service_00000000001E0000.bin
```

```
Cleaning C:\...\restart-service_00000000001E0000.bin
Renaming all obfuscated symbols
Saving ...\restart-service_00000000001E0000-cleaned.bin
```

Drag the `-cleaned.bin` file onto dnSpy. The full decompiled C# source is now readable, revealing the binary is a custom `runas.exe` wrapper that restarts an Oracle service using hardcoded credentials embedded directly in the source. 

Those credentials can now be tested against other services in the environment for lateral movement.
