# Windows Security

Windows security is a **large and complex topic** because the operating system itself is large and complex. As you work with Windows systems, you will quickly realise that the attack surface is broad. Even fully patched systems can be vulnerable if they are **poorly configured** or if security features are misunderstood or misapplied 

Microsoft has significantly improved Windows security over the years. As environments have become more interconnected and attackers more capable, additional defensive layers have been introduced. Your job is to understand **how these mechanisms work**, **what assumptions they make**, and **where they fail**.

---

## Windows Security Model (High Level)

Windows enforces security through a combination of:

* Identities (users, computers, services)
* Permissions and access checks
* Tokens and authentication
* Centralised policy enforcement
* Monitoring and auditing

Every action taken on a Windows system is evaluated against this model before it is allowed to execute.

---

## Security Identifiers (SID)

Every **security principal** in Windows is identified by a **Security Identifier (SID)**.

Security principals include:

* Users
* Groups
* Computers
* Services
* Processes and threads

SIDs are **unique** and generated automatically by the system. Even if two users have the same username, Windows treats them as completely different entities if their SIDs differ.

When you authenticate, your SID (and the SIDs of any groups you belong to) are embedded into your **access token**. This token defines **everything you are allowed to do** on the system.

---

### Viewing Your SID

You can view your current SID using:

```text
whoami /user
```

Example output:

```text
User Name   SID
ws01\bob   S-1-5-21-674899381-4069889467-2080702030-1002
```

---

### SID Structure

A SID follows a predictable structure:

```text
S-1-5-21-674899381-4069889467-2080702030-1002
```

Breaking it down:

| Component                         | Meaning                        |
| --------------------------------- | ------------------------------ |
| `S`                               | Identifies the string as a SID |
| `1`                               | Revision level (always 1)      |
| `5`                               | Identifier authority           |
| `21`                              | Indicates a user or group SID  |
| `674899381-4069889467-2080702030` | Domain or machine identifier   |
| `1002`                            | Relative ID (RID)              |

The **RID** is especially important because it distinguishes:

* Normal users
* Administrators
* Guest accounts
* Built-in service accounts

---

## Security Accounts Manager (SAM) and ACLs

Windows permissions are enforced using **Access Control Lists (ACLs)**, which are made up of **Access Control Entries (ACEs)**.

ACLs define:

* Who can access an object
* What actions are allowed
* Under which conditions

Every securable object (files, registry keys, services, processes) has a **security descriptor**, which contains:

* Owner
* Group
* **DACL** (who is allowed or denied access)
* **SACL** (what actions are audited)

---

### Access Tokens and LSA

Every process you start runs with an **access token** issued by the **Local Security Authority (LSA)**.

That token contains:

* Your SID
* Group SIDs
* Privilege assignments
* Integrity level

Understanding how tokens are built and validated is critical when you later study **privilege escalation**.

---

## User Account Control (UAC)

**User Account Control (UAC)** exists to prevent silent privilege escalation.

You have already seen it:

* Consent prompts
* Password requests
* Elevation confirmations

UAC introduces **Admin Approval Mode**, where even administrators operate with **limited tokens** by default. Elevated privileges are only granted after explicit approval.

Key points you should remember:

* UAC is not a sandbox
* UAC is not malware protection
* UAC is a friction mechanism

It slows down attackers and malware but does not stop them outright.

---

## Windows Registry

The **Windows Registry** is a hierarchical database that stores configuration for:

* The operating system
* Installed applications
* User preferences

You can open it with:

```text
regedit
```

---

### Registry Structure

The registry is organised into **root keys**, such as:

* `HKEY_LOCAL_MACHINE (HKLM)` – system-wide settings
* `HKEY_CURRENT_USER (HKCU)` – user-specific settings

HKLM contains subkeys such as:

* `SAM`
* `SECURITY`
* `SYSTEM`
* `SOFTWARE`
* `HARDWARE`
* `BCD`

Most of these are loaded into memory at boot.

---

### Registry Storage on Disk

The system-wide registry hives are stored under:

```text
C:\Windows\System32\Config\
```

Examples include:

* `SAM`
* `SECURITY`
* `SYSTEM`
* `SOFTWARE`

User-specific registry data is stored in:

```text
C:\Users\<USERNAME>\NTUSER.DAT
```

This file backs **HKCU** for that user.

---

## Run and RunOnce Registry Keys

Certain registry keys are evaluated automatically when:

* The system boots
* A user logs in

These keys are commonly used for **persistence**.

Important locations:

```text
HKLM\Software\Microsoft\Windows\CurrentVersion\Run
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce
HKCU\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

If you see writable values here, you should pay attention.

---

## Application Whitelisting

Application whitelisting works on a **default-deny** model.

Instead of blocking known bad software, it allows **only explicitly approved software** to run.

Key points:

* Reduces attack surface
* Prevents unauthorised tools
* Requires careful planning

Whitelisting is recommended by organisations such as **NIST**, especially in high-security environments.

---

## AppLocker

**AppLocker** is Microsoft’s application whitelisting solution.

It allows administrators to control:

* Executables
* Scripts
* MSI installers
* DLLs
* Packaged applications

Rules can be based on:

* Digital signatures
* File paths
* Hashes
* User or group membership

AppLocker is usually deployed in **audit mode first**, then enforced once administrators are confident it will not disrupt business operations.

---

## Local Group Policy

**Group Policy** allows administrators to centrally control system behaviour.

Even on standalone machines, **Local Group Policy** can enforce:

* Password policies
* Application restrictions
* Audit settings
* Security hardening

You can open the editor with:

```text
gpedit.msc
```

It is divided into:

* Computer Configuration
* User Configuration

Many security features, including **Credential Guard** and **AppLocker**, are configured through Group Policy.

---

## Windows Defender Antivirus

**Windows Defender Antivirus** is built into modern Windows systems and enabled by default.

It provides:

* Real-time protection
* Behaviour monitoring
* Cloud-based analysis
* Tamper Protection

You can inspect Defender status with:

```text
Get-MpComputerStatus
```

Defender performs very well in detection tests and integrates tightly with the OS.

However:

* It is not a silver bullet
* It should be part of a layered defence
* Misconfigurations weaken it significantly

Defender reliably detects common frameworks and unmodified tools such as **Metasploit payloads** and **Mimikatz**.

---

## Final Notes for You

Windows security is built on **layers**, not absolutes.

As you continue:

* Learn how each mechanism works in isolation
* Then learn how they interact
* Finally, learn how attackers abuse assumptions between them

Misconfigurations matter more than missing patches.

If you understand **SIDs, tokens, ACLs, UAC, the registry, and policy enforcement**, Windows security will stop feeling opaque and start feeling predictable.
