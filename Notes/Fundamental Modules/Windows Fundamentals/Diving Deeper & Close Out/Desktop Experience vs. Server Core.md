# Desktop Experience vs. Server Core

When you deploy Windows Server, one of the most important early decisions is whether to install **Desktop Experience** or **Server Core**. This choice has long-term consequences for **security, manageability, performance, and attack surface**, and it cannot be reversed after installation.

You should understand both models, not just from an administrative perspective, but also from an **offensive and defensive security standpoint**.

---

## What Is Server Core?

**Server Core** was introduced with **Windows Server 2008** as a **minimal installation option**. It includes only the components required to run essential server roles, deliberately excluding most graphical elements.

The main goals of Server Core are:

* Reduced attack surface
* Lower resource usage (CPU, memory, disk)
* Fewer updates and reboots
* Simplified long-term maintenance

Because it lacks a full GUI, Server Core forces you to manage the system using:

* Command Prompt
* PowerShell
* Remote management tools (MMC, RSAT, WinRM)

---

## Desktop Experience

**Desktop Experience** is the traditional Windows Server installation most people are familiar with.

It includes:

* Full graphical user interface
* Windows Explorer
* Control Panel
* Server Manager
* MMC snap-ins
* Browser support

This makes it easier to manage, especially for administrators who rely heavily on GUI-based workflows.

---

## Management Differences

### Server Core Management

On Server Core:

* All local administration is command-line driven
* PowerShell is the primary management interface
* GUI-based administration is performed remotely
* Initial setup is handled using **`sconfig`**

`sconfig` is a text-based configuration utility (implemented as a VBScript) that allows you to:

* Configure networking
* Join a domain
* Manage accounts
* Enable remote management
* Configure Windows Update
* Activate Windows

This is typically the **first interface you see** after installation.

---

### Desktop Experience Management

With Desktop Experience:

* Server Manager launches automatically
* MMC snap-ins are available locally
* GUI-based troubleshooting is straightforward
* Less reliance on remote tooling

This lowers the learning curve but increases complexity and exposure.

---

## GUI Support on Server Core

Although Server Core does not include a full GUI, **some graphical tools are still available**.

Supported tools include:

* Registry Editor (`regedit`)
* Notepad
* System Information
* Windows Installer
* Task Manager
* PowerShell

It also supports selected **Sysinternals tools**, such as:

* Process Explorer
* Process Monitor
* TCPView
* Active Directory Explorer

This partial GUI support is intentional and focused on diagnostics.

---

## Installation Constraints

Starting with **Windows Server 2019**, you must choose:

* **Server Core**, or
* **Desktop Experience**

at **installation time**.

You **cannot switch** between them later. This makes planning essential.

---

## Application Limitations on Server Core

Not all server applications are supported on Server Core.

Common examples that **cannot run** on Server Core include:

* System Center Virtual Machine Manager (SCVMM) 2019
* System Center Data Protection Manager 2019
* SharePoint Server 2019
* Project Server 2019

This often dictates the installation choice before security considerations are even evaluated.

---

## Feature Comparison

Below is a high-level comparison of commonly used components.

| Application / Feature   | Server Core   | Desktop Experience |
| ----------------------- | ------------- | ------------------ |
| Command Prompt          | Available     | Available          |
| PowerShell / .NET       | Available     | Available          |
| Registry Editor         | Available     | Available          |
| Disk Management         | Not Available | Available          |
| Server Manager          | Not Available | Available          |
| MMC                     | Not Available | Available          |
| Event Viewer            | Not Available | Available          |
| Services Console        | Not Available | Available          |
| Control Panel           | Not Available | Available          |
| Windows Explorer        | Not Available | Available          |
| Task Manager            | Available     | Available          |
| Web Browser             | Not Available | Available          |
| Remote Desktop Services | Available     | Available          |

---

## Security Perspective

From a security point of view, Server Core offers clear advantages:

* Fewer running services
* Fewer installed components
* Smaller attack surface
* Reduced opportunities for GUI-based abuse

However, it also:

* Requires stronger command-line skills
* Makes local troubleshooting more difficult
* Increases reliance on remote management

Desktop Experience is easier to manage but inherently exposes more functionality and more potential attack vectors.

---

## Choosing Between the Two

The decision should be based on:

* Business requirements
* Server role and workload
* Administrator skill level
* Security posture
* Operational maturity

In high-security or high-scale environments, Server Core is often preferred. In smaller environments or where GUI-based tools are required, Desktop Experience may be more practical.

---

## Key Takeaways

* Server Core is minimal, lightweight, and security-focused
* Desktop Experience is feature-rich and easier to manage
* The choice is permanent after installation
* Server Core reduces attack surface but increases complexity
* Desktop Experience trades security for convenience

As you encounter Windows servers in the field, always take note of **which installation type you are dealing with**. It influences how you enumerate, how you exploit, and how you defend the system.
