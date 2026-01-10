# Active Directory Terminology

Before you go any further, it is important to align on terminology. Active Directory has its own vocabulary, and you will see these terms repeatedly when reading documentation, using tools, or analysing attacks. The sections below define the key concepts you need to be comfortable with when working in or attacking an Active Directory environment.

---

## Object

An **object** is any resource stored in Active Directory. If it exists inside the directory, it is an object.

Common examples include:

* Users
* Computers
* Groups
* Organisational Units (OUs)
* Printers
* Domain Controllers

Everything you interact with in AD is represented as an object.

---

## Attributes

Every object in Active Directory is described by a set of **attributes**. Attributes define the properties and characteristics of an object.

For example:

* A user object has attributes such as:

  * givenName
  * displayName
  * sAMAccountName
* A computer object has attributes such as:

  * hostname
  * DNS name

Each attribute has an associated **LDAP name**, which is what you use when querying Active Directory programmatically or through tools.

---

## Schema

The **Active Directory schema** defines what can exist in the directory and how it is structured. You can think of it as the blueprint for the entire environment.

The schema:

* Defines object classes (for example, user, computer, group)
* Defines which attributes belong to each class
* Specifies which attributes are required and which are optional

When an object is created from a class, this is called **instantiation**, and the resulting object is an **instance** of that class.

For example:

* The class is `computer`
* The object `RDS01` is an instance of that class

---

## Domain

A **domain** is a logical boundary that groups together objects such as:

* Users
* Computers
* Groups
* OUs

Domains can exist independently or be connected to other domains using trust relationships. From a security perspective, the domain is a critical boundary for authentication and policy enforcement.

---

## Forest

A **forest** is the top-level container in Active Directory.

Key points:

* A forest contains one or more domains
* It represents the primary security boundary
* All objects inside a forest share a common schema

You can think of a forest as the largest administrative scope within Active Directory.

---

## Tree

A **tree** is a collection of domains that share a common namespace and start from a single root domain.

Important characteristics:

* Domains in a tree share a contiguous DNS namespace
* Parent-child trusts are created automatically
* All domains in a tree use a shared Global Catalog

A forest can contain multiple trees, but trees in the same forest cannot share the same namespace.

---

## Container

A **container** is an object that can hold other objects.

Examples include:

* Domains
* OUs
* Built-in containers

Containers define where objects live in the directory hierarchy.

---

## Leaf

A **leaf object** is an object that does not contain other objects.

Examples:

* Users
* Computers
* Groups

Leaf objects sit at the end of the directory structure.

---

## Global Unique Identifier (GUID)

A **GUID** is a 128-bit value assigned to every object when it is created.

Key characteristics:

* Stored in the `objectGUID` attribute
* Globally unique across the enterprise
* Never changes for the lifetime of the object

Active Directory uses GUIDs internally to track objects. Searching by GUID is one of the most reliable ways to identify a specific object, especially when names are duplicated or changed.

---

## Security Principals

A **security principal** is anything that can be authenticated and assigned permissions.

This includes:

* Domain users
* Computer accounts
* Service accounts
* Processes running under a user or computer context

Only security principals can be granted access to resources. Local users and groups are managed separately by the Security Accounts Manager (SAM) and are not AD objects.

---

## Security Identifier (SID)

A **SID** uniquely identifies a security principal or security group.

Important points:

* Issued by a Domain Controller
* Unique and never reused
* Stored in access tokens during logon

When a user logs in, their access token contains:

* Their SID
* The SIDs of all groups they belong to
* Their assigned privileges

SIDs are what the system actually checks when enforcing access.

---

## Distinguished Name (DN)

A **Distinguished Name (DN)** defines the full path to an object in Active Directory.

Example:

```
cn=bjones,ou=IT,ou=Employees,dc=inlanefreight,dc=local
```

This DN describes:

* The user `bjones`
* Located in the IT OU
* Inside the Employees OU
* Within the inlanefreight.local domain

DNs must be unique across the directory.

---

## Relative Distinguished Name (RDN)

The **Relative Distinguished Name (RDN)** is the part of the DN that uniquely identifies an object within its parent container.

In the previous example:

* `cn=bjones` is the RDN

Objects can share the same RDN as long as their full DN is different.

<img width="2576" height="1579" alt="dn_rdn2" src="https://github.com/user-attachments/assets/aaf53c6c-3b2f-4106-81fc-5c3e48f3e900" />

---

## sAMAccountName

The **sAMAccountName** is the legacy logon name.

Characteristics:

* Typically short (for example, `bjones`)
* Must be unique within the domain
* Limited to 20 characters

This value is still heavily used for authentication and tooling.

---

## userPrincipalName (UPN)

The **userPrincipalName** is an alternative logon identifier.

Format:

```
username@domain
```

Example:

```
bjones@inlanefreight.local
```

This attribute is optional but commonly used in modern environments.

---

## FSMO Roles

**Flexible Single Master Operation (FSMO)** roles exist to prevent conflicts in a multi-DC environment.

There are five FSMO roles:

Forest-wide:

* Schema Master
* Domain Naming Master

Domain-wide:

* RID Master
* PDC Emulator
* Infrastructure Master

These roles distribute responsibility across DCs and prevent replication issues. They can be transferred if needed and are critical to domain stability.

---

## Global Catalog (GC)

A **Global Catalog** is a Domain Controller configured to store:

* A full copy of objects from its own domain
* A partial copy of objects from all other domains in the forest

The GC is used for:

* Logon authentication (group membership resolution)
* Searching for objects across the entire forest

---

## Read-Only Domain Controller (RODC)

An **RODC** hosts a read-only copy of the Active Directory database.

Key properties:

* Does not store user passwords by default
* Does not replicate changes to other DCs
* Includes read-only DNS
* Reduces risk in remote or untrusted locations

---

## Replication

**Replication** keeps Domain Controllers synchronised.

How it works:

* Changes on one DC are replicated to others
* The Knowledge Consistency Checker (KCC) manages connections
* Ensures availability and redundancy

Replication failures often lead to serious security and stability issues.

---

## Service Principal Name (SPN)

An **SPN** uniquely identifies a service instance.

SPNs are used by Kerberos to:

* Map services to accounts
* Allow clients to authenticate without knowing the service account name

Misconfigured SPNs are commonly abused in Kerberos-based attacks.

---

## Group Policy Object (GPO)

A **GPO** is a collection of policy settings.

GPOs can:

* Apply to users or computers
* Be linked at domain or OU level
* Control security settings, scripts, and system behaviour

Each GPO is identified by a unique GUID.

---

## Access Control List (ACL)

An **ACL** defines who can access an object and how.

It is made up of multiple **Access Control Entries (ACEs)**.

---

## Access Control Entry (ACE)

An **ACE** specifies:

* A trustee (user or group)
* Allowed, denied, or audited permissions

ACEs are processed in order to determine access.

---

## Discretionary Access Control List (DACL)

A **DACL** controls access to an object.

Important behaviour:

* No DACL means full access for everyone
* An empty DACL denies all access
* ACEs are evaluated sequentially

---

## System Access Control List (SACL)

A **SACL** defines which actions should be audited.

It controls logging of access attempts in the Security Event Log.

---

## Fully Qualified Domain Name (FQDN)

An **FQDN** uniquely identifies a host using DNS.

Format:

```
hostname.domain.tld
```

Example:

```
DC01.INLANEFREIGHT.LOCAL
```

---

## Tombstone

A **tombstone** represents a deleted AD object.

Key behaviour:

* Object is marked as deleted
* Most attributes are stripped
* Retained for the tombstone lifetime

After this period, the object is permanently removed.

---

## AD Recycle Bin

The **AD Recycle Bin** allows full recovery of deleted objects.

Advantages:

* Preserves most attributes
* No need to restore backups
* Default recovery window of 60 days

---

## SYSVOL

**SYSVOL** stores shared domain data such as:

* Group Policy files
* Logon and logoff scripts

It is replicated to all DCs and is a frequent attack target.

---

## AdminSDHolder

**AdminSDHolder** enforces protected permissions on privileged accounts.

Key points:

* Applies a protected ACL
* Enforced by the SDProp process
* Runs by default every hour

This prevents persistent modification of privileged accounts.

---

## dsHeuristics

The **dsHeuristics** attribute controls forest-wide behaviour.

One common use:

* Excluding groups from AdminSDHolder protection

Misuse can weaken privilege protection.

---

## adminCount

The **adminCount** attribute indicates whether an object is protected.

* `1` means protected
* Often marks privileged accounts

Attackers frequently target these accounts.

---

## Active Directory Users and Computers (ADUC)

**ADUC** is the primary GUI for managing:

* Users
* Groups
* Computers

Most actions here can also be performed via PowerShell.

---

## ADSI Edit

**ADSI Edit** provides low-level access to Active Directory.

Capabilities include:

* Modifying any attribute
* Creating or deleting objects

This tool is powerful and dangerous if misused.

---

## sIDHistory

The **sIDHistory** attribute stores previous SIDs.

Used for:

* Domain migrations

If abused, it can allow privilege escalation.

---

## NTDS.DIT

**NTDS.DIT** is the Active Directory database.

It contains:

* All directory objects
* Password hashes

Compromise of this file usually means full domain compromise.

---

## MSBROWSE

**MSBROWSE** is a legacy browsing protocol.

* Used in older Windows networks
* Largely obsolete today
* Replaced by SMB and modern discovery mechanisms

You will rarely encounter it in modern environments, but it may appear during legacy assessments.
