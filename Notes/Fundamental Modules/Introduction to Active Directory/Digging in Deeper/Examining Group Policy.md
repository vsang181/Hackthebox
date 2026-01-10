## Examining Group Policy

Group Policy is a Windows feature that allows administrators to centrally manage and enforce configuration settings across users and computers. Every Windows system includes a **Local Group Policy Editor**, but in enterprise environments the real power comes from **domain-based Group Policy** integrated with Active Directory.

From a security perspective, Group Policy is one of the most effective mechanisms for shaping and enforcing an organisation’s security posture at scale. Active Directory is not secure by default, and properly designed Group Policy configurations form a critical layer in any defense-in-depth strategy.

At the same time, Group Policy represents a high-impact attack surface. If an attacker gains sufficient permissions over a Group Policy Object (GPO), it can be abused for:

* Privilege escalation
* Lateral movement
* Persistence
* Full domain compromise

Understanding how Group Policy works is therefore essential for both defenders and penetration testers.

---

## Group Policy Objects (GPOs)

A **Group Policy Object (GPO)** is a logical container that holds one or more policy settings. These settings can apply to:

* Users
* Computers
* Or both, depending on configuration

Each GPO:

* Has a unique name
* Is identified internally by a globally unique identifier (GUID)
* Can be linked to one or more AD containers (sites, domains, or OUs)

A single container can have multiple GPOs applied, and a single GPO can be linked to multiple containers.

Common examples of what GPOs can control include:

* Password and account lockout policies
* Screen lock and inactivity timeouts
* USB and removable media usage
* Application execution (CMD, PowerShell, scripts)
* Audit and logging configuration
* Software deployment
* Login banners
* Kerberos and NTLM behaviour
* Startup, shutdown, logon, and logoff scripts

---

## Example GPO Use Cases

Typical security-relevant uses of GPOs include:

* Enforcing different password policies for:

  * Standard users
  * Administrators
  * Service accounts
* Blocking removable storage devices
* Enforcing password-protected screen savers
* Restricting access to command-line tools
* Enabling advanced audit logging
* Blocking execution of unauthorised software
* Deploying software across the domain
* Preventing installation of unapproved applications
* Displaying legal or monitoring banners at logon
* Disabling legacy authentication mechanisms such as LM hashes
* Executing scripts during system or user lifecycle events

---

## Default Password Policy Example (Windows Server 2008)

In a default Windows Server 2008 domain, password complexity is enabled with the following requirements:

* Minimum length: **7 characters**
* Must include characters from at least **three of four** categories:

  * Uppercase letters (A–Z)
  * Lowercase letters (a–z)
  * Numbers (0–9)
  * Special characters

While functional, this baseline is insufficient against modern offline cracking and password spraying attacks and should be strengthened via Group Policy.

---

## GPO Processing and Order of Precedence

GPOs are applied according to a strict hierarchy. When multiple policies define the same setting, the policy applied last takes precedence.

### Order of Precedence

1. **Local Group Policy**
2. **Site-level GPOs**
3. **Domain-level GPOs**
4. **Organisational Unit (OU) GPOs**
5. **Nested OU GPOs**

<img width="2576" height="1316" alt="gpo_levels" src="https://github.com/user-attachments/assets/f00e9b71-a041-4c16-ac41-e59e07ec3b33" />

Important rules:

* OU-level policies override domain-level policies
* Domain-level policies override site-level policies
* Site-level policies override local policies
* **Computer configuration settings override user configuration settings** when conflicts exist

---

## Managing Group Policy

Group Policy can be managed using:

* Group Policy Management Console (GPMC)
* PowerShell (GroupPolicy module)
* Custom administrative tooling

Two default GPOs are created automatically when a domain is set up:

* **Default Domain Policy**
* **Default Domain Controllers Policy**

Best practice is to:

* Use the Default Domain Policy only for truly domain-wide baseline settings
* Avoid adding unrelated or experimental settings to it
* Create separate, purpose-specific GPOs for role-based or security-specific controls

---

## GPO Link Order

When multiple GPOs are linked to the same container:

* GPOs are processed based on **Link Order**
* The GPO with **Link Order 1** has the **highest precedence**
* Higher numbered GPOs are processed earlier and can be overridden

This makes link order a critical factor when troubleshooting unexpected policy behaviour.

---

## Enforced GPOs (No Override)

A GPO can be marked as **Enforced**.

When enforced:

* Its settings cannot be overridden by GPOs linked at lower levels
* This applies regardless of OU hierarchy

If the **Default Domain Policy** is enforced, it will override all other GPOs across the domain.

---

## Block Inheritance

An OU can be configured to **Block Inheritance**.

Effects:

* GPOs from parent containers (such as the domain or higher-level OUs) will not apply
* Locally linked GPOs will still apply

Important rule:

* **Enforced GPOs always override Block Inheritance**

---

## Group Policy Refresh Interval

Group Policy is not applied instantly.

Default refresh behaviour:

* Workstations and servers: every **90 minutes ± 30 minutes**
* Domain Controllers: every **5 minutes**

This random offset prevents all systems from requesting policies simultaneously.

Manual refresh:

```
gpupdate /force
```

Refresh intervals can be modified via Group Policy, but overly aggressive intervals may cause:

* Network congestion
* Replication delays
* Domain controller performance issues

---

## Security Considerations of GPOs

GPOs are a high-value target for attackers.

If an attacker can modify a GPO that applies to:

* A privileged user
* A sensitive server
* A domain controller

They may be able to:

* Add users to privileged groups
* Grant local administrator rights
* Create scheduled tasks
* Execute malicious scripts
* Deploy malware at scale
* Establish long-term persistence

Tools such as BloodHound are commonly used to identify:

* GPOs with weak permissions
* Misconfigured delegation
* Attack paths involving GPO modification rights

Mismanaged Group Policy is one of the most common causes of full Active Directory compromise in real-world environments.
