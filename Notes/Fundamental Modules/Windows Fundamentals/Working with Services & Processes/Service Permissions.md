# Service Permissions

Windows services are long-running background processes and form a **core part of the operating system**. From a security perspective, they are also one of the **most commonly overlooked attack surfaces**. Poorly configured service permissions can allow you to load malicious DLLs, execute arbitrary programs, escalate privileges, or maintain persistence without needing full administrative access 

Many of these weaknesses are introduced unintentionally, often by **third-party software installers** or by administrators making small mistakes during setup.

Your first task is not exploitation, but **awareness**. Service permissions exist, they matter, and they are frequently misconfigured.

---

## Why Service Accounts Matter

When a service is installed on Windows, it must run under a specific **user context**. By default, this context is often inherited from the account that performed the installation.

Consider this scenario:

* You log on to a server as **Bob**
* You install a network service (for example, DHCP)
* You do not explicitly change the service account
* The service is now configured to run as **Bob**

This creates multiple problems.

If Bob later leaves the organisation and his account is disabled, **every service running as Bob will fail to start**. In the case of DHCP, that means:

* Clients cannot obtain IP addresses
* Network connectivity fails
* Productivity is lost

This is not just a security issue, but an **availability issue**.

### Service Accounts

Best practice is to use **dedicated service accounts**:

* Created solely to run a specific service
* Granted only the permissions they require
* Not used for interactive logons

These accounts reduce risk and align with the **principle of least privilege**.

---

## Services as an Attack Vector

When assessing services, you must consider **two separate permission surfaces**:

1. **The service configuration itself**
2. **The file system permissions of the executable path**

If either is weak, the service may be exploitable.

For example:

* If you can modify the service binary path
* Or replace the executable or DLL it loads

You can often execute code with the service’s privileges, which may be **LocalSystem**.

---

## Examining Services with `services.msc`

The graphical Services console (`services.msc`) allows you to view and manage almost every aspect of a service.

Using **Windows Update (`wuauserv`)** as an example, you can inspect:

* Service name
* Display name
* Description
* Startup type
* Current status
* Log on account
* Executable path

### Why the Executable Path Matters

The **Path to executable** defines exactly what is launched when the service starts.

If the directory containing that executable has **weak NTFS permissions**, an attacker may be able to:

* Replace the executable
* Drop a malicious DLL
* Hijack execution

This directly ties service security to NTFS permissions.

---

## Service Logon Accounts

Most services run as **LocalSystem** by default, which is the **highest privilege level** on a standalone Windows system.

Built-in service accounts include:

* **LocalSystem** – full local privileges
* **LocalService** – limited local privileges
* **NetworkService** – limited local privileges with network access

Not all services need LocalSystem access. You should always question whether that level of privilege is justified.

---

## Service Recovery Options

The **Recovery** tab in a service’s properties is often ignored, but it can be dangerous.

A service can be configured to:

* Restart itself
* Restart the system
* **Run a program on failure**

If an attacker can modify this setting or the referenced program, they can abuse a legitimate service to execute malicious code.

---

## Examining Services with `sc.exe`

The `sc` utility is a command-line interface to the Service Control Manager and is far more suitable for scripting and large environments.

### Querying a Service Configuration

```text
sc qc wuauserv
```

This reveals:

* Service type
* Startup behaviour
* Binary path
* Dependencies
* Logon account

Knowing the **service name** is essential when working from the command line.

You can also query services remotely:

```text
sc \\HOSTNAME query ServiceName
```

---

### Starting and Stopping Services

```text
sc stop wuauserv
```

If you see **Access is denied**, you are not running with sufficient privileges. This is expected behaviour.

---

### Modifying a Service Binary Path

```text
sc config wuauserv binPath= C:\Winbows\Perfectlylegitprogram.exe
```

If this succeeds and the service runs as **LocalSystem**, you have effectively achieved **privileged code execution**.

This is why writable service configurations are so dangerous.

---

## Inspecting Service Security Descriptors

You can inspect detailed service permissions using:

```text
sc sdshow wuauserv
```

The output is expressed in **SDDL (Security Descriptor Definition Language)**. It looks intimidating, but it follows strict rules.

Every securable object in Windows has:

* An owner
* A primary group
* A **DACL** (who can do what)
* A **SACL** (what gets audited)

In security testing, you usually care about the **DACL**.

---

## Reading SDDL (High-Level View)

Example fragment:

```text
D:(A;;CCLCSWRPLORC;;;AU)
```

You can interpret this as:

* **D:** → DACL
* **A** → Access allowed
* **AU** → Authenticated Users
* **CC, LC, SW, RP, LO, RC** → Specific service permissions

Each two-letter code maps to a specific action, such as:

* Querying configuration
* Querying status
* Starting the service
* Reading the security descriptor

Once you realise that SDDL is just a compact ACL, it becomes much easier to reason about.

---

## Examining Service Permissions with PowerShell

PowerShell provides a more readable way to inspect service permissions by querying the registry.

```text
Get-ACL -Path HKLM:\System\CurrentControlSet\Services\wuauserv | Format-List
```

This output shows:

* Owner and group
* Allowed permissions
* Security principals
* SDDL representation
* Associated SIDs

Unlike `sc`, PowerShell exposes **full identity information**, which is extremely useful during analysis.

---

## Why Command-Line Access Matters

GUI tools are helpful for learning, but they **do not scale**.

In real environments, you need to:

* Enumerate hundreds of systems
* Script checks for weak permissions
* Analyse services remotely
* Automate reporting

Understanding services from the command line gives you speed, consistency, and visibility.

---

## Key Takeaways

* Services are a high-value attack surface
* Misconfigured service accounts cause both security and availability issues
* Writable service binaries or configurations often lead to privilege escalation
* `sc.exe` is script-friendly and powerful
* SDDL looks complex, but it is just structured ACL data
* PowerShell provides clearer visibility than GUI tools

When you learn to read service permissions confidently, Windows services stop being “background noise” and start becoming one of your most reliable sources of findings.
