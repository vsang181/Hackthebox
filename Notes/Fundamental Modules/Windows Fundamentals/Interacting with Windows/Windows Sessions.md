# Windows Sessions

When you work with Windows systems, every action happens inside a **session**. Understanding the different session types helps you reason about **who is executing code**, **with what privileges**, and **how access was obtained**. This distinction becomes critical when you analyse logons, services, scheduled tasks, and privilege escalation paths.

---

## Interactive Sessions

An **interactive session** is created when a user actively authenticates to a system by providing credentials.

You will encounter interactive logons in several common scenarios:

* Logging in locally at the physical machine
* Logging in remotely using **Remote Desktop (RDP)**
* Starting a secondary session with the `runas` command
* Authenticating with a domain account on a domain-joined system

In all of these cases, a real user is involved, credentials are supplied, and the session is tied to that user’s security context.

From a security perspective, interactive sessions are important because they:

* Generate detailed authentication logs
* Create user tokens tied to group membership
* Expose user-specific artefacts (profiles, AppData, history)
* Are often the starting point for lateral movement

Whenever you gain access as a real user, you are operating inside an interactive session.

---

## Non-Interactive Sessions

**Non-interactive sessions** are fundamentally different. These sessions do not require a user to log in or supply credentials. Instead, they are used by Windows itself to run background components automatically.

Non-interactive accounts are primarily used to:

* Start services at boot time
* Run background applications
* Execute scheduled tasks
* Perform operating system maintenance

These accounts **do not have passwords** in the traditional sense and are not meant for human interaction.

Windows provides **three built-in non-interactive accounts**.

---

## Types of Non-Interactive Accounts

### Local System Account

Also known as **`NT AUTHORITY\SYSTEM`**, this is the **most powerful account on a Windows system**.

Key characteristics:

* Higher privileges than local administrators
* Full control over the local machine
* Used by core operating system services
* Can access almost all system resources

If you achieve code execution as SYSTEM, you effectively own the machine.

---

### Local Service Account

Known as **`NT AUTHORITY\LocalService`**, this account is a **restricted version of SYSTEM**.

Key characteristics:

* Limited local privileges
* Similar access level to a standard local user
* Used for services that do not require high privileges
* Designed to reduce damage if compromised

This account exists to follow the principle of least privilege.

---

### Network Service Account

Known as **`NT AUTHORITY\NetworkService`**, this account is designed for services that need **network access**.

Key characteristics:

* Limited local privileges, similar to Local Service
* Can authenticate to other systems over the network
* Appears as the computer account when accessing domain resources
* Commonly used by network-facing services

This account is especially relevant when analysing **lateral movement** and service-to-service authentication.

---

## Comparing the Non-Interactive Accounts

| Account         | Local Privileges | Network Capabilities         | Typical Use             |
| --------------- | ---------------- | ---------------------------- | ----------------------- |
| Local System    | Very high        | Full                         | Core OS services        |
| Local Service   | Low              | None                         | Low-risk local services |
| Network Service | Low              | Authenticated network access | Network-facing services |

---

## Why This Matters to You

As a penetration tester, you must always ask:

* *Which account is this process running as?*
* *Is this session interactive or non-interactive?*
* *What privileges does this context actually have?*

Many real-world privilege escalations happen because:

* A service runs as SYSTEM when it does not need to
* A non-interactive account has excessive file permissions
* A scheduled task runs under a powerful context
* Administrators assume “no user logged in” means “no risk”

Once you understand how Windows sessions and service accounts work, it becomes much easier to identify **where trust boundaries are broken** and **where escalation paths exist**.
