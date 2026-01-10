# Security in Active Directory

As you progress through Active Directory, you start to see a clear pattern: AD is designed to make information sharing and centralised management easy at scale. That convenience comes at a cost. In many ways, Active Directory can be described as insecure by default. A fresh AD deployment lacks most of the hardening controls required to withstand modern attacks unless they are deliberately configured.

From a security perspective, Active Directory leans heavily towards **availability** and **confidentiality**, often at the expense of **integrity**. This creates a large attack surface where misconfigurations, excessive privileges, and weak operational practices can be chained together to achieve full domain compromise.

A secure AD environment is not achieved through a single control or product. It requires layered defences, consistent monitoring, and disciplined administration. The measures below represent baseline hardening practices that you should expect to see in any reasonably secured enterprise domain.

## General Active Directory Hardening Measures

### Local Administrator Password Solution (LAPS)

Microsoft LAPS is used to automatically randomise and rotate local administrator passwords on domain-joined Windows hosts.

Key benefits include:

* Prevents password reuse across machines
* Limits lateral movement after a single host compromise
* Centralises password storage in Active Directory with access controls

Password rotation intervals are configurable and should be aligned with organisational risk tolerance. LAPS should be treated as a supporting control, not a standalone solution.

### Audit Policy Settings (Logging and Monitoring)

Logging and monitoring are essential for detecting abuse, misconfigurations, and active attacks.

Effective auditing allows you to identify:

* Account creation and modification noted in AD
* Password changes and resets
* Abnormal authentication patterns such as password spraying
* Privilege escalation attempts
* Kerberos abuse and ticket manipulation
* Enumeration activity targeting AD objects

Without meaningful logs, attackers can operate undetected for long periods.

### Group Policy Security Settings

Group Policy Objects (GPOs) are one of the most powerful security mechanisms in Active Directory. When used correctly, they allow consistent enforcement of security controls across users and computers.

Common security-related policy categories include:

* **Account Policies**

  * Password policies
  * Account lockout thresholds
  * Kerberos ticket lifetimes

* **Local Policies**

  * User rights assignments
  * Security auditing
  * Control over system-level actions such as driver installation and removable media usage

* **Software Restriction Policies**

  * Prevent execution of unauthorised software

* **Application Control Policies**

  * Restrict execution of scripts, installers, and binaries
  * Often implemented using AppLocker
  * Frequently applied to block CMD and PowerShell for non-administrative users

* **Advanced Audit Policy Configuration**

  * Granular auditing of file access, privilege usage, policy changes, and authentication events

These controls are not foolproof but significantly raise the cost of successful attacks when combined.

<img width="1410" height="727" alt="adv-audit-pol" src="https://github.com/user-attachments/assets/2afb6633-38ee-4af1-9c89-c94aa9038c69" />

### Update Management (WSUS / SCCM)

Patch management is critical in Windows and AD-heavy environments.

* **WSUS** provides centralised patch approval and deployment
* **SCCM** builds on WSUS and offers advanced reporting, targeting, and automation

Unpatched domain controllers and servers remain one of the most common causes of full domain compromise.

### Group Managed Service Accounts (gMSA)

Group Managed Service Accounts provide secure, domain-managed credentials for services and scheduled tasks.

Advantages include:

* Automatically generated 120-character passwords
* Automatic password rotation
* No need for human knowledge of credentials
* Safe usage across multiple hosts

gMSAs should be preferred over traditional service accounts wherever supported.

### Security Groups

Security groups allow you to assign permissions and privileges at scale.

Instead of assigning rights directly to users:

* Assign permissions to groups
* Add users to groups based on role
* Review group membership regularly

This approach simplifies audits and reduces configuration errors.

### Account Separation

Administrators should always use separate accounts for administrative and non-administrative tasks.

A typical model:

* Standard user account for email, browsing, and document work
* Privileged account used only on secured administrative systems

Passwords must be unique across accounts to reduce credential reuse risk.

### Password Policies, Passphrases, and MFA

Short or predictable passwords are one of the easiest attack vectors in Active Directory.

Recommended practices:

* Minimum length of at least 12 characters for standard users
* Longer passwords for administrators and service accounts
* Use passphrases or password managers
* Implement password filters to block common patterns and organisation-specific terms
* Enforce multi-factor authentication for remote access, especially RDP

Complexity alone is insufficient without length and monitoring.

### Limiting Domain Admin Usage

Domain Admin credentials should only be used on:

* Domain Controllers
* Dedicated administrative workstations

They should never be used on:

* Personal workstations
* Web servers
* Application servers
* Jump hosts used by multiple users

This reduces credential exposure in memory and limits attack paths.

### Auditing and Removing Stale Objects

Unused accounts and systems increase risk.

Regularly review:

* Disabled but not deleted user accounts
* Legacy service accounts
* Old computer objects
* Accounts with weak or never-rotated passwords

Unused objects should be removed or securely disabled.

### Auditing Permissions and Access

Permissions drift over time.

You should periodically audit:

* Local administrator rights
* Membership in Domain Admins and Enterprise Admins
* Access to file shares and sensitive systems
* Privileged group memberships

Excessive privilege is one of the most common enterprise security failures.

### Audit Policies and Logging

Visibility is non-negotiable.

Well-configured logging enables detection of:

* Password spraying attempts
* Kerberoasting activity
* AD enumeration
* Privilege abuse
* Suspicious authentication behaviour

Security teams should align audit policies with Microsoft’s recommended baselines.

### Restricted Groups

Restricted Groups allow administrators to enforce group membership via Group Policy.

Common use cases include:

* Enforcing local administrator membership on endpoints
* Locking down Enterprise Admins and Schema Admins
* Preventing unauthorised privilege escalation

This ensures group membership does not drift over time.

### Limiting Server Roles

Sensitive systems should have minimal roles installed.

Examples of poor practice include:

* Hosting IIS on a Domain Controller
* Running web applications on Exchange servers
* Combining database and application roles on a single host

Role separation reduces attack surface and blast radius.

### Limiting Local Administrator and RDP Rights

Local admin and RDP access should be tightly controlled.

Common issues include:

* Domain Users added as local admins
* Excessive RDP access to shared servers

Either mistake allows a single compromised account to escalate quickly and pivot across the environment.

Strong control of these permissions dramatically reduces lateral movement opportunities.
