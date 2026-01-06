# Operating System Structure (Windows)

When you work on a Windows system, everything starts from the **root of a drive**, written as `<drive_letter>:\`. In most environments, the operating system lives on the **C:** drive, which is also called the **boot partition**. Any additional physical or virtual disks are assigned other letters, such as **D:** or **E:**.

As you enumerate a Windows host, understanding this layout helps you quickly identify where applications, user data, configuration files, and security-relevant artefacts are likely to live.

---

## Core Directories on the System Drive

Below is a breakdown of the most important directories you will encounter on a typical Windows installation and why they matter to you.

### `PerfLogs`

This directory can store Windows performance logs. It is usually empty by default but may contain diagnostic data on systems where performance monitoring has been enabled.

---

### `Program Files`

On **32-bit systems**, this directory contains both 16-bit and 32-bit applications.
On **64-bit systems**, it contains **only 64-bit applications**.

Installed software, third-party tools, and enterprise applications commonly live here. During assessments, this is a good place to look for outdated or misconfigured software.

---

### `Program Files (x86)`

This directory exists only on **64-bit systems** and is used for **32-bit and 16-bit applications**.

The separation between `Program Files` and `Program Files (x86)` helps Windows manage compatibility, but it also helps you quickly identify which architecture an application is using.

---

### `ProgramData`

This is a **hidden directory** that stores application data shared across all users on the system.

Programs use this directory to store configuration files, databases, caches, and logs that must be accessible regardless of which user is logged in. From a security perspective, this folder often contains interesting artefacts.

---

### `Users`

This directory contains **user profiles** for every account that has logged on to the system.

Inside it, you will typically find:

* Individual user folders
* `Public`
* `Default`

This is one of the most important directories during local enumeration.

---

### `Default`

The **template profile** used whenever a new user account is created. New user profiles are generated based on the contents of this directory.

Misconfigurations here can affect every new user created on the system.

---

### `Public`

A shared folder intended for file sharing between users.

By default:

* All local users can access it
* It is shared over the network
* Network access still requires valid credentials

You should always check this directory for accidentally exposed files.

---

### `AppData`

Each user profile contains a hidden `AppData` directory that stores **per-user application data and settings**.

You will encounter three important subdirectories here:

* **Roaming**
  Contains machine-independent data that follows the user profile, such as settings and custom dictionaries.

* **Local**
  Stores machine-specific data that is not synchronised across systems.

* **LocalLow**
  Similar to `Local`, but with a lower integrity level. It is often used by applications running in restricted modes, such as web browsers.

Application credentials, tokens, and configuration files are frequently stored here.

---

### `Windows`

This directory contains the majority of the files required for the operating system to function.

You should treat this directory with caution. Modifying files here can destabilise or break the system.

---

### `System`, `System32`, `SysWOW64`

These directories contain **core Windows DLLs and binaries** required by the operating system and the Windows API.

When an application loads a DLL without specifying a full path, Windows searches these directories first. This behaviour is important when you later study **DLL search order hijacking**.

---

### `WinSxS`

The **Windows Component Store**.

This directory contains:

* Copies of Windows components
* Updates and service packs
* Versioned system files

Although it looks extremely large, it plays a critical role in system updates and recovery.

---

## Exploring Directories from the Command Line

You do not need a graphical interface to explore the Windows file system. The command line is often faster and more precise.

### Listing Directory Contents

You can use the `dir` command to list files and directories.

```text
dir C:\ /a
```

The `/a` flag ensures that hidden and system files are included in the output.

This command gives you immediate visibility into:

* System files
* Page and hibernation files
* Core directories
* Available free space

---

## Visualising Directory Trees

When you want to understand the structure of a directory and its subfolders, the `tree` command is extremely useful.

### Example: Viewing a Program’s Directory Structure

```text
tree "C:\Program Files (x86)\VMware"
```

This produces a hierarchical view of:

* Subdirectories
* Application components
* Language files
* Tools and binaries

This is helpful when you are trying to understand how an application is organised or where configuration files might be stored.

---

### Walking an Entire Drive

To recursively list all files and directories on a drive, one screen at a time:

```text
tree C:\ /f | more
```

* `/f` includes files, not just directories
* `more` prevents the output from scrolling too fast

You can apply this command to **any directory**, not just the root of the drive.

---

## Why This Matters

As you continue working with Windows systems, you should aim to **recognise these directories instantly**. When you know where things usually live, enumeration becomes structured instead of random.

Treat the file system as a map. The better you know it, the faster you can move—and the easier it becomes to spot what does not belong.
