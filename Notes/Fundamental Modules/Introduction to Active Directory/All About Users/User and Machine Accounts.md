# User and Machine Accounts

If you want to understand Active Directory properly, you need to get comfortable with how **accounts** work. Accounts are how Windows decides who you are, what you can access, and what you are allowed to do.

An account can represent:

* A person (employee, contractor, administrator)
* A service or application running in the background
* A computer that is joined to the domain

When you log on, Windows validates your credentials and then creates an **access token**. This token is attached to your session and is presented whenever you interact with the operating system. It contains your security identity and group membership, and it is what Windows actually uses to make access-control decisions.

<img width="1020" height="703" alt="all_users" src="https://github.com/user-attachments/assets/745ef183-03f5-423a-89d0-d96f3a1b3ed9" />

In practice, accounts exist to:

* Let users sign in and work normally
* Run services under a defined security context
* Control access to files, shares, printers, and applications
* Simplify access management through group membership

If you assign permissions to individuals one-by-one, administration becomes painful. If you assign permissions to groups, you can manage membership and let permissions flow naturally.

---

## Why user accounts matter in Active Directory

Provisioning and maintaining user accounts is one of the core jobs Active Directory performs. In most organisations, you should expect at least one domain user account per person. In many cases, you will see multiple accounts per person, for example:

* A standard daily-use account
* A privileged IT/admin account
* A separate account for specific operational tasks

You will also see service accounts used to run applications, scheduled tasks, and domain services. This can create surprisingly large numbers. An organisation with 1,000 employees may easily have far more than 1,000 active accounts once you include service accounts, admin accounts, and automation accounts.

You will also often find large numbers of disabled accounts. Organisations commonly disable accounts rather than delete them, especially where auditing or record retention is required. In many environments, these accounts are moved into a dedicated OU such as:

* FORMER EMPLOYEES
* DISABLED USERS
* LEAVERS

This matters because forgotten accounts and poorly reviewed group memberships create real risk.

Active Directory Users and Computers interface showing organizational units and security groups under INLANEFREIGHT.LOCAL, including Employees, Security Groups, and Service Accounts.

---

## Privileges and misconfiguration risk

User accounts can be configured across a very wide spectrum, from:

* A basic domain user with mostly read access
* To highly privileged roles with broad administrative control

Because rights and permissions are so flexible, they are also easy to get wrong. Over-permissioning, stale group membership, and misapplied delegated rights are all common issues. During real assessments, user accounts are often a primary starting point because:

* Credentials are frequently exposed through phishing or weak password practices
* Users install unauthorised software
* Administrators make overly permissive changes under pressure

If you want to reduce risk, you must rely on defence in depth, including:

* Strong password policies and enforced MFA where possible
* Least privilege and careful delegation
* Monitoring for unusual logons and privilege changes
* Regular review of group memberships and account hygiene

This section is not a deep dive into every user-focused attack technique, but you should take away one key idea: **accounts are one of the largest and most abused attack surfaces in Active Directory**.

---

## Local Accounts

Local accounts exist only on a specific workstation or server. They are stored on that host and can only be granted rights on that same host.

Key points you should remember:

* Local accounts are security principals
* Their access does not automatically apply across the domain
* They are managed locally, not by Active Directory

Windows creates several default local accounts.

### Administrator

* First local account created during Windows installation
* Has full control over almost all resources on the host
* Cannot be deleted or locked out
* Can be renamed or disabled
* Modern Windows versions commonly disable it by default and create a separate local administrator during setup

### Guest

* Disabled by default
* Intended for temporary access with minimal privileges
* Historically had a blank password
* Recommended to remain disabled due to obvious risk

### SYSTEM

* Appears as `NT AUTHORITY\SYSTEM`
* Used by the operating system to run core services and processes
* Has the highest level of access on a Windows host
* Does not have a standard user profile
* Does not appear in typical user management tools
* Cannot be added to groups
* Has broad permissions across the local system by default

### Network Service

* Used by the Service Control Manager to run services
* When accessing remote services, it can present credentials externally

### Local Service

* Also used by the Service Control Manager
* Runs with minimal local privileges
* Presents anonymous credentials to the network

You should understand how these accounts behave because misconfigurations involving services and local privileges frequently lead to escalation paths.

---

## Domain Users

Domain users differ from local users because the domain grants them rights to access resources across the environment.

Domain user accounts can typically:

* Authenticate to domain services
* Access shared resources (if permitted)
* Log in to domain-joined machines (depending on policy)

Permissions come from:

* Direct rights assigned to the user
* Group memberships
* Policies applied through Group Policy

### KRBTGT account

One account you must always be aware of is **KRBTGT**. It is built into the AD infrastructure and is used by the Key Distribution Centre for Kerberos ticketing.

Why you should care:

* Compromise of KRBTGT enables extremely powerful Kerberos abuse
* It is central to attacks such as Golden Ticket
* It can be used for privilege escalation and long-term persistence

---

## User Naming Attributes

Active Directory includes multiple attributes used to identify and reference user accounts. You should know the common ones because tools and scripts will use them constantly.

### Common naming attributes

* **UserPrincipalName (UPN)**

  * Primary logon identifier in the form `user@domain`
  * Commonly aligned with email address conventions

* **ObjectGUID**

  * Permanent unique identifier for the object
  * Does not change during the object’s lifetime

* **sAMAccountName**

  * Legacy logon name used broadly across Windows tooling

* **objectSID**

  * The security identifier for the account
  * Used heavily during authorisation decisions

* **sIDHistory**

  * Stores previous SIDs after migrations
  * Can become a security issue if poorly controlled

### Example user attributes

User and Machine Accounts

```
PS C:\htb Get-ADUser -Identity htb-student

DistinguishedName : CN=htb student,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
Enabled           : True
GivenName         : htb
Name              : htb student
ObjectClass       : user
ObjectGUID        : aa799587-c641-4c23-a2f7-75850b4dd7e3
SamAccountName    : htb-student
SID               : S-1-5-21-3842939050-3880317879-2865463114-1111
Surname           : student
UserPrincipalName : htb-student@INLANEFREIGHT.LOCAL
```

Many attributes exist that you will never use, but you should familiarise yourself with the ones that often contain useful or sensitive information, such as descriptions, manager fields, email addresses, and custom attributes used by internal systems.

---

## Domain-joined vs Non-domain-joined Machines

Not every Windows machine is managed the same way. You need to understand the difference between domain-managed hosts and workgroup hosts because it affects both security controls and attack options.

### Domain-joined

A domain-joined host:

* Pulls policy and configuration from the domain
* Is managed centrally (commonly through Group Policy)
* Allows a domain user to log in on multiple domain-joined machines
* Fits the standard enterprise model

This is what you should expect to see in most corporate environments.

### Non-domain-joined (workgroup)

A workgroup host:

* Is not managed by domain policy
* Uses only local users and local groups
* Does not share identity information with the domain
* Is typically used in homes or small standalone networks

Users manage changes locally, and accounts do not roam between machines.

---

## Why SYSTEM matters in a domain

Here is a point that many people miss during early AD work: if you gain `NT AUTHORITY\SYSTEM` on a domain-joined machine, you often gain an effective launchpad into the domain.

In many environments, the machine account context provides rights similar to a standard domain user. This means you may be able to:

* Enumerate large portions of Active Directory
* Query users, groups, computers, and policies
* Identify misconfigurations and attack paths

You do not always need to compromise an individual user account first. If you obtain SYSTEM on a domain-joined host through exploitation or local privilege escalation, you can often begin meaningful domain reconnaissance immediately.
