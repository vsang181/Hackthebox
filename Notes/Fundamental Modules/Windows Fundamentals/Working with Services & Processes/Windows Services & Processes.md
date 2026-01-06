# Windows Services and Processes

Windows relies heavily on **services and processes** to function. As a penetration tester, you must understand how these components work, how they are managed, and where misconfigurations commonly appear. Poorly configured services are one of the **most reliable privilege escalation vectors** on Windows systems 

This section focuses on how services differ from regular processes, how to enumerate them, and why some of them are especially high-value during assessments.

---

## Windows Services

A **Windows service** is a long-running process designed to operate in the background. Unlike standard applications, services:

* Can start automatically at system boot
* Do not require a user to be logged in
* Continue running after a user logs out
* Often run with elevated privileges

Many core Windows functions rely on services, including:

* Networking
* Credential management
* Windows Update
* Logging and diagnostics
* Security enforcement

Applications can also install themselves as services. For example, antivirus software, monitoring agents, backup tools, and management software commonly run as services.

---

## Service Control Manager (SCM)

All Windows services are managed by the **Service Control Manager (SCM)**.

You can interact with the SCM using:

* **GUI:** `services.msc` (MMC snap-in)
* **Command line:** `sc.exe`
* **PowerShell:** `Get-Service`, `Set-Service`, `Start-Service`, `Stop-Service`

The Services management console shows key details such as:

* Service name
* Display name
* Description
* Current status
* Startup type
* Account the service runs under

---

## Enumerating Services with PowerShell

A simple way to list running services is:

```text
Get-Service | ? {$_.Status -eq "Running"} | Select -First 2 | fl
```

Example output:

```text
Name                : AdobeARMservice
DisplayName         : Adobe Acrobat Update Service
Status              : Running
CanStop             : True
ServiceType         : Win32OwnProcess

Name                : Appinfo
DisplayName         : Application Information
Status              : Running
ServiceType         : Win32OwnProcess, Win32ShareProcess
```

This tells you:

* Whether the service is running
* Whether it can be stopped
* How it is implemented

These details matter when analysing attack surface.

---

## Service States and Startup Types

Services can be in several states:

* **Running**
* **Stopped**
* **Paused**
* **Starting**
* **Stopping**

They can also be configured to start:

* **Automatically**
* **Automatically (Delayed)**
* **Manually**
* **Disabled**

Misconfigured startup settings are often abused to achieve persistence or privilege escalation.

---

## Service Categories

Windows services typically fall into three broad categories:

* **Local Services** – run with limited local privileges
* **Network Services** – interact with network resources
* **System Services** – core services with high privileges

Most services can only be created, modified, or deleted by administrators. However, **incorrect permissions on existing services** are a common vulnerability.

---

## Critical Windows System Services

Some services and processes are so critical that stopping them will destabilise or crash the system. These generally cannot be safely restarted without a reboot.

| Process                  | Purpose                            |
| ------------------------ | ---------------------------------- |
| `smss.exe`               | Session Manager Subsystem          |
| `csrss.exe`              | Client Server Runtime              |
| `wininit.exe`            | Initialises system processes       |
| `logonui.exe`            | Handles user logon                 |
| `lsass.exe`              | Authentication and security policy |
| `services.exe`           | Starts and stops services          |
| `winlogon.exe`           | Secure logon and session control   |
| `System`                 | Windows kernel process             |
| `svchost.exe (RPCSS)`    | Hosts critical DLL-based services  |
| `svchost.exe (DCOM/PnP)` | Distributed COM and Plug and Play  |

You should never attempt to terminate these during testing unless you fully understand the impact.

---

## Processes on Windows

A **process** is an instance of a running program. Processes may be:

* Part of the operating system
* Spawned by services
* Started by users
* Launched by applications

Unlike services, many user-level processes can be terminated without serious consequences. Others, especially system processes, are critical.

Examples of high-impact processes include:

* Windows Logon Application
* Client Server Runtime
* Windows Session Manager
* Service Host (`svchost.exe`)
* Local Security Authority Subsystem Service (`lsass.exe`)

---

## Local Security Authority Subsystem Service (LSASS)

`lsass.exe` is one of the **most valuable targets** on a Windows system.

LSASS is responsible for:

* Verifying user logon attempts
* Creating access tokens
* Enforcing security policy
* Handling password changes

All authentication events involving LSASS are logged in the **Windows Security Log**.

Because LSASS stores credentials in memory, it is heavily targeted by attackers. Numerous tools exist to extract **hashed and cleartext credentials** from LSASS if sufficient privileges are obtained.

---

## Sysinternals Tools

The **Sysinternals Suite** is a collection of powerful, portable tools for Windows system analysis and administration. Most tools do not require installation.

You can access them directly from Microsoft’s live share:

```text
\\live.sysinternals.com\tools
```

For example, running ProcDump directly:

```text
\\live.sysinternals.com\tools\procdump.exe -accepteula
```

This avoids writing tools to disk, which can help evade basic detection mechanisms.

---

### Commonly Used Sysinternals Tools

* **Process Explorer** – advanced Task Manager replacement
* **Process Monitor** – file system, registry, and network monitoring
* **TCPView** – active network connections
* **PSExec** – remote command execution over SMB

For penetration testers, these tools are invaluable for:

* Discovering interesting processes
* Analysing service behaviour
* Identifying privilege escalation paths
* Supporting lateral movement

---

## Task Manager

**Task Manager** is a built-in Windows utility that provides visibility into:

* Running processes
* Services
* Startup programs
* Logged-in users
* System performance

You can open it using:

* `Ctrl + Shift + Esc`
* `Ctrl + Alt + Del` → Task Manager
* `taskmgr` from CMD or PowerShell

---

### Key Task Manager Tabs

| Tab         | Purpose                                       |
| ----------- | --------------------------------------------- |
| Processes   | Running applications and background processes |
| Performance | CPU, memory, disk, network, GPU usage         |
| App history | Per-application resource usage                |
| Startup     | Programs that run at boot                     |
| Users       | Logged-in users and their processes           |
| Details     | PID, user context, CPU and memory usage       |
| Services    | Installed services and their status           |

From the **Performance** tab, you can also open **Resource Monitor** for deeper visibility into CPU, memory, disk, and network activity.

---

## Process Explorer (Sysinternals)

**Process Explorer** goes far beyond Task Manager.

It allows you to:

* View parent-child process relationships
* Inspect loaded DLLs
* Analyse open handles
* Identify orphaned processes
* Search for specific handles or DLLs across the system

This is extremely useful when investigating suspicious behaviour or understanding how applications interact with the system.

---

## Why This Matters

Windows services and processes form the backbone of the operating system. From an attacker’s perspective, they also represent:

* Privilege boundaries
* Persistence mechanisms
* Credential storage locations
* Lateral movement opportunities

If you can confidently enumerate services, understand what they do, and spot misconfigurations, you gain a **massive advantage** during Windows assessments.

Always pay attention to:

* What runs automatically
* What runs as SYSTEM
* What can be modified by non-administrators

That is where real-world Windows compromises usually begin.
