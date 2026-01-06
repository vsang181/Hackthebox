# Windows Management Instrumentation (WMI)

Windows Management Instrumentation (WMI) is a **core Windows subsystem** that gives you deep visibility and control over both local and remote systems. As you work through Windows environments, you will see WMI everywhere—sometimes explicitly, sometimes hidden behind higher-level tools. Understanding it properly gives you a major advantage during enumeration, automation, and lateral movement 

WMI has been included by default in Windows since **Windows 2000**, and its original goal was to **standardise management and monitoring** across large corporate networks.

---

## What WMI Is (Conceptually)

You can think of WMI as a **management layer** that sits between the operating system and management tools.

Instead of every tool talking directly to the OS in its own way, WMI provides a consistent interface to:

* Query system information
* Modify configuration
* Start or stop processes
* Interact with hardware and software components
* Perform actions on remote machines

This is why both administrators and attackers rely on it.

---

## Core WMI Components

WMI is made up of several tightly connected components. You do not need to memorise these, but you **must understand how they relate**.

### WMI Service

The Windows Management Instrumentation service runs automatically at boot. It acts as the **broker** between applications, providers, and the WMI database.

---

### Managed Objects

These represent **anything that can be managed**, such as:

* Processes
* Services
* Operating system properties
* Hardware devices
* Network interfaces

If it can be queried or controlled, it is usually exposed as a managed object.

---

### WMI Providers

Providers are responsible for **collecting data** about a specific object type and reporting it to WMI.

Each provider focuses on a particular area, such as:

* Processes
* Disks
* Network adapters
* Event logs

---

### Classes

Classes define **what data exists** and **how it is structured**.

Examples:

* `Win32_OperatingSystem`
* `Win32_Process`
* `Win32_Service`

Classes are the entry point for almost every WMI query you will write.

---

### Methods

Methods are actions attached to classes.

They allow you to:

* Start or stop processes
* Rename files
* Modify configuration
* Trigger system behaviour

Methods are especially powerful because they enable **remote execution** without needing traditional remote shells.

---

### WMI Repository

This is a **local database** that stores static WMI data, such as class definitions and metadata.

---

### CIM Object Manager (CIMOM)

The CIM Object Manager receives requests from tools and:

1. Talks to the appropriate provider
2. Collects the requested data
3. Returns it to the requesting application

You usually never interact with CIMOM directly, but everything passes through it.

---

### WMI API

The API allows applications and scripts to communicate with the WMI infrastructure.

---

### WMI Consumer

A consumer is any application or script that **requests data or actions** through WMI.

PowerShell, WMIC, and many enterprise tools act as WMI consumers.

---

## What You Can Do with WMI

WMI is extremely flexible. Using it, you can:

* Query status information from local or remote systems
* Configure security settings
* Modify user and group permissions
* Change system properties
* Execute code
* Schedule processes
* Configure logging and monitoring

This dual-use nature is why WMI is equally valuable to **blue teams and red teams**.

---

## WMIC (WMI Command-Line Interface)

WMIC is the legacy command-line interface for WMI. It can be used either:

* Interactively
* As a one-liner from CMD

To open the interactive shell:

```text
wmic
```

To run a direct command:

```text
wmic computersystem get name
```

This returns the system hostname.

---

### WMIC Help and Deprecation

You can view available commands and switches with:

```text
wmic /?
```

You will notice a warning that **WMIC is deprecated**. This means:

* It is no longer being actively developed
* It still exists on many systems
* It still works

In modern environments, PowerShell is preferred, but WMIC remains useful on older systems and restricted shells.

---

### Example: Querying OS Information with WMIC

```text
wmic os list brief
```

Example output:

```text
BuildNumber  Organization  RegisteredUser  SerialNumber             SystemDirectory      Version
19041                      Owner           00123-00123-00123-AAOEM  C:\Windows\system32  10.0.19041
```

WMIC uses:

* **Aliases** (e.g. `os`)
* **Verbs** (e.g. `list`)
* **Adverbs** (e.g. `brief`)

This structure is consistent across WMIC commands.

---

## Using WMI with PowerShell

PowerShell provides a much more flexible and readable way to interact with WMI.

The most commonly used cmdlet is:

```text
Get-WmiObject
```

This cmdlet retrieves instances of WMI classes from local or remote systems.

---

### Example: Querying Operating System Details

```text
Get-WmiObject -Class Win32_OperatingSystem |
select SystemDirectory,BuildNumber,SerialNumber,Version |
ft
```

Example output:

```text
SystemDirectory     BuildNumber SerialNumber            Version
---------------     ----------- ------------            -------
C:\Windows\system32 19041       00123-00123-00123-AAOEM 10.0.19041
```

This approach gives you structured objects instead of raw text, which is far easier to filter, sort, and automate.

---

## Invoking WMI Methods

To **perform actions**, not just queries, you can use:

```text
Invoke-WmiMethod
```

### Example: Renaming a File via WMI

```text
Invoke-WmiMethod -Path "CIM_DataFile.Name='C:\Users\Public\spns.csv'" `
-Name Rename `
-ArgumentList "C:\Users\Public\kerberoasted_users.csv"
```

Example output:

```text
ReturnValue : 0
```

A return value of `0` indicates success.

This example demonstrates an important point:
**You are interacting with the file system through WMI, not traditional commands**.

---

## Why WMI Matters for You

WMI is powerful because it:

* Works locally and remotely
* Is trusted by the operating system
* Is heavily used in enterprise environments
* Often bypasses simplistic security controls

Later, you will see WMI used for:

* Enumeration
* Remote command execution
* Lateral movement
* Persistence

If you understand WMI early, many advanced techniques will feel logical instead of mysterious.

---

## Key Takeaways

* WMI is a built-in Windows management framework
* Everything in WMI revolves around classes, methods, and providers
* WMIC is deprecated but still useful in legacy scenarios
* PowerShell provides cleaner and more powerful WMI interaction
* WMI is a favourite tool for both administrators and attackers

Treat WMI as **infrastructure**, not just a tool. Once you understand how it works, you will start seeing it everywhere in Windows environments.
