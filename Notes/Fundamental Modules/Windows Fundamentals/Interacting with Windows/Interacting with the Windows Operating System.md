# Interacting with the Windows Operating System

Windows provides multiple ways for you to interact with the operating system. In practice, you will switch between **graphical interfaces** and **command-line tools** depending on the task, your access level, and how stealthy or efficient you need to be. As a penetration tester, you should be comfortable with **all of these interaction methods**, not just one 

---

## Graphical User Interface (GUI)

The **graphical user interface (GUI)** was introduced to make computers accessible to users who were not comfortable with command-line interaction. Early research at Xerox PARC in the late 1970s heavily influenced how GUIs later appeared in Apple and Microsoft operating systems.

A GUI allows you to interact with the system using:

* Windows
* Icons
* Menus
* Mouse-based input

Most everyday Windows users never need to touch the command line. They rely entirely on point-and-click interaction for browsing files, running applications, and changing settings.

From an administrative perspective, GUIs are still heavily used for tasks such as:

* Managing **Active Directory**
* Configuring **IIS**
* Viewing **event logs**
* Managing **local users and groups**

During assessments, GUI access via RDP often gives you fast situational awareness.

---

## Remote Desktop Protocol (RDP)

**Remote Desktop Protocol (RDP)** is a proprietary Microsoft protocol that allows you to connect to a remote Windows system and interact with its GUI as if you were sitting in front of it.

Key characteristics you should remember:

* Uses **TCP port 3389** by default
* Follows a **client–server model**
* Requires an RDP client on your machine
* Requires the RDP service enabled on the target

When you connect via RDP, you receive a **full interactive desktop session**, identical to a local logon. This makes RDP extremely popular with system administrators for remote management.

In real environments, RDP is commonly used:

* By administrators managing servers
* By employees working remotely via VPN
* As a jump point between internal systems

From an offensive point of view, exposed or weakly protected RDP services are high-value targets.

---

## Windows Command Line

While the GUI is convenient, the **command line gives you far more control**. It allows you to automate tasks, work faster, and operate in environments where GUI access is restricted or unavailable.

On Windows, you primarily interact with the system through:

* **Command Prompt (CMD)**
* **PowerShell**

You should treat both as essential tools.

---

## Command Prompt (CMD)

The **Command Prompt (`cmd.exe`)** is the traditional Windows command-line interface.

You can launch it by:

* Searching for `cmd` in the Start Menu
* Typing `cmd` in the Run dialogue
* Executing `C:\Windows\System32\cmd.exe` directly

CMD is commonly used for:

* Quick one-off commands (`ipconfig`, `whoami`)
* Batch files
* Scheduled tasks
* Basic system administration

To see available commands, you can type:

```text
help
```

For help on a specific command:

```text
help <command>
```

Many commands also support inline help using:

```text
<command> /?
```

For example:

```text
ipconfig /?
```

CMD is simple, predictable, and still widely used, especially in legacy environments.

---

## PowerShell

**PowerShell** was designed to go beyond CMD and give administrators deep access to the operating system.

Key characteristics:

* Built on the **.NET framework**
* Object-oriented (not just text-based output)
* Designed for automation and administration
* Supports scripting, modules, and pipelines

PowerShell gives you access to:

* The file system
* The registry
* Windows services
* Processes
* Event logs
* WMI and CIM

In modern Windows environments, PowerShell is often **the most powerful interface you have**.

---

## Cmdlets

PowerShell uses **cmdlets**, which are small, focused commands.

Cmdlets follow a strict naming convention:

```text
Verb-Noun
```

Examples:

* `Get-ChildItem`
* `Set-Location`
* `Get-Process`
* `Get-Service`

Cmdlets:

* Accept parameters and flags
* Support tab-completion
* Return structured objects

For example, to recursively list files:

```text
Get-ChildItem -Recurse
```

Or to list another directory:

```text
Get-ChildItem -Path C:\Users\Administrator\Documents
```

Once you understand how cmdlets are structured, discovery becomes much faster.

---

## Aliases

PowerShell supports **aliases**, which are short names for cmdlets.

Examples:

* `cd` → `Set-Location`
* `ls` → `Get-ChildItem`
* `cat` → `Get-Content`

You can view all aliases with:

```text
Get-Alias
```

You can also create your own:

```text
New-Alias -Name Show-Files Get-ChildItem
```

Aliases improve speed but can reduce clarity in scripts, so use them carefully.

---

## PowerShell Help System

PowerShell includes a built-in help system for:

* Cmdlets
* Functions
* Scripts
* Concepts

You can access it using:

```text
Get-Help <cmdlet>
```

By default, help files are not installed locally. You can:

* View online help with `-Online`
* Download help files using `Update-Help`

Without installed help files, PowerShell shows partial information only.

---

## Running PowerShell Scripts

PowerShell scripts (`.ps1`) allow you to automate complex tasks.

You can:

* Run scripts locally
* Import scripts into memory
* Load modules to expose new functions

For example, importing a script so its functions are available:

```text
Import-Module .\PowerView.ps1
```

Once imported, you can list loaded modules:

```text
Get-Module
```

This approach is commonly used during enumeration and post-exploitation.

---

## Execution Policy

PowerShell includes a feature called the **execution policy**, which is designed to prevent accidental execution of untrusted scripts. It is **not a security boundary**.

Common execution policies include:

* **Restricted** – scripts cannot run
* **RemoteSigned** – downloaded scripts must be signed
* **Bypass** – no restrictions
* **AllSigned** – all scripts must be signed
* **Unrestricted** – warnings but scripts can run

You can view current policies with:

```text
Get-ExecutionPolicy -List
```

You can change the policy for the current session only:

```text
Set-ExecutionPolicy Bypass -Scope Process
```

This change:

* Requires minimal privileges
* Applies only to the current PowerShell session
* Is commonly used during assessments

This is why execution policy should **never be treated as a defensive control**.

---

## What You Should Take Away

As you work through Windows environments, you should be able to:

* Switch comfortably between GUI, CMD, and PowerShell
* Understand when each interface is most appropriate
* Use PowerShell for structured enumeration and automation
* Recognise execution policy limitations
* Read help output instead of guessing syntax

Windows interaction is not about preference. It is about **choosing the right tool for the situation**.
