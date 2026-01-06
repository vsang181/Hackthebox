# Windows File Systems and Permissions

Windows supports several file systems, each designed for different use cases. As you enumerate Windows hosts, you should understand **which file system is in use**, **what features it provides**, and **how permissions are enforced**, because these details directly affect security, persistence, and privilege escalation paths 

For modern environments, your primary focus should always be **NTFS**.

---

## Windows File System Types

Windows supports five main file systems:

* FAT12
* FAT16
* FAT32
* NTFS
* exFAT

FAT12 and FAT16 are obsolete and no longer relevant in modern systems. In practice, you will mainly encounter **FAT32**, **exFAT**, and **NTFS**.

---

## FAT32

**FAT32 (File Allocation Table 32)** is still widely used, especially on removable media.

The “32” refers to the use of 32-bit values to track data clusters on disk.

### Where You Will See FAT32

* USB flash drives
* SD cards
* External storage
* Occasionally older or compatibility-focused systems

### Advantages of FAT32

* Excellent device compatibility
* Supported by Windows, Linux, and macOS
* Works across cameras, consoles, phones, and embedded systems

### Limitations of FAT32

* Maximum file size of **4 GB**
* No built-in file permissions
* No native encryption or compression
* Requires third-party tools for security controls

From a security perspective, FAT32 offers **no access control**. Any user with access to the device can read or modify its contents.

---

## NTFS

**NTFS (New Technology File System)** has been the default Windows file system since **Windows NT 3.1**.

NTFS addresses the shortcomings of FAT-based systems and introduces features critical for enterprise security and reliability.

### Advantages of NTFS

* Built-in journaling to recover from crashes or power loss
* Support for very large files and partitions
* Rich metadata support
* Fine-grained file and folder permissions
* Better performance due to improved data structures

### Limitations of NTFS

* Limited native support on mobile and embedded devices
* Older media devices (TVs, cameras) often cannot read NTFS

In real-world assessments, almost every internal Windows system you touch will be using NTFS.

---

## NTFS Permissions Overview

NTFS enforces access control using **Access Control Lists (ACLs)**. These determine **who can access what**, and **how**.

Below are the most important permission types you should recognise.

| Permission           | Description                                         |
| -------------------- | --------------------------------------------------- |
| Full Control         | Read, write, modify, and delete files or folders    |
| Modify               | Read, write, and delete                             |
| List Folder Contents | View and list folders and subfolders (folders only) |
| Read and Execute     | View contents and execute files                     |
| Write                | Create files and modify existing ones               |
| Read                 | View file contents and directory listings           |
| Traverse Folder      | Move through directories without listing them       |

### Traverse Folder in Practice

A user might not be able to list a directory’s contents, but with **Traverse Folder** permission, they can still access a file if they know its full path.

This behaviour is often overlooked and can be abused during enumeration.

---

## Permission Inheritance

By default, NTFS permissions are **inherited** from parent directories. This greatly simplifies administration, as permissions do not need to be set on every individual file.

Inheritance can be:

* Enabled (default behaviour)
* Disabled for specific files or folders

Disabling inheritance allows administrators to apply **explicit permissions**, but it also increases complexity and the risk of misconfiguration.

---

## Managing Permissions with `icacls`

NTFS permissions can be managed through the GUI, but during assessments and automation, you will rely heavily on the **`icacls`** command-line utility.

---

### Viewing Permissions

You can list permissions on a directory like this:

```text
icacls C:\Windows
```

Example output (trimmed for clarity):

```text
c:\windows NT SERVICE\TrustedInstaller:(F)
           NT AUTHORITY\SYSTEM:(OI)(CI)(IO)(F)
           BUILTIN\Administrators:(M)
           BUILTIN\Users:(RX)
```

Each entry shows:

* The user or group
* The access level
* Inheritance flags

---

## Inheritance Flags Explained

You will often see these flags in `icacls` output:

* **(CI)** – Container inherit (applies to subdirectories)
* **(OI)** – Object inherit (applies to files)
* **(IO)** – Inherit only (does not apply to the current object)
* **(NP)** – Do not propagate inheritance
* **(I)** – Inherited from a parent directory

Understanding these flags is essential when you are analysing why a user can or cannot access something.

---

## Basic Access Rights Codes

`icacls` uses shorthand codes to represent access levels:

* **F** – Full access
* **M** – Modify
* **RX** – Read and execute
* **R** – Read-only
* **W** – Write-only
* **D** – Delete
* **N** – No access

---

## Modifying Permissions from the Command Line

You can grant permissions like this:

```text
icacls C:\Users /grant joe:F
```

This gives the user `joe` full control over **only the `C:\Users` directory itself**.

Because `(OI)` and `(CI)` were not specified:

* The permission does **not** apply to subfolders
* The permission does **not** apply to files inside

This distinction is critical and often misunderstood.

---

### Verifying the Change

```text
icacls C:\Users
```

You will now see an entry similar to:

```text
WS01\joe:(F)
```

---

### Removing Permissions

To revoke access:

```text
icacls C:\Users /remove joe
```

---

## Why `icacls` Matters

`icacls` is extremely powerful. It allows you to:

* Grant or revoke permissions
* Explicitly deny access
* Enable or disable inheritance
* Change file or directory ownership
* Apply permissions across large directory trees

In domain environments, misconfigured NTFS permissions are a **common source of privilege escalation**. Learning to read `icacls` output fluently will save you a huge amount of time during assessments.

---

## Key Takeaway

NTFS is not just a file system. It is a **security boundary**.

If you understand:

* How NTFS permissions work
* How inheritance behaves
* How to read and manipulate ACLs

you gain a major advantage when enumerating Windows systems. Always treat file permissions as potential attack surfaces, not just administrative details.
