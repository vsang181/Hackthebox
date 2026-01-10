# Active Directory Structure

When you work with **Active Directory**, you are dealing with Microsoft’s directory service for Windows-based networks. It is designed as a **distributed, hierarchical system** that centralises control over identities, systems, and resources across an organisation. In most enterprise environments, Active Directory becomes the backbone for how access is granted, enforced, and audited.

At a high level, Active Directory is responsible for:

* Authentication within a Windows domain
* Authorisation to access systems and resources
* Centralised management of identities and devices

Behind the scenes, this functionality is provided by **Active Directory Domain Services (AD DS)**. AD DS stores directory data and exposes it to both administrators and standard users. This includes information such as user accounts, passwords, group memberships, computer objects, and the permissions that govern access to them.

Active Directory was introduced with Windows Server 2000 and has evolved continuously since then. A key design goal has always been backward compatibility. While this makes upgrades and legacy integration easier, it also means that many features rely on correct configuration rather than being secure by default. In large environments, this complexity makes Active Directory difficult to manage and easy to misconfigure.

From an attack perspective, these misconfigurations are extremely valuable. Weak Active Directory design is often abused to:

* Gain an initial internal foothold
* Move laterally between systems
* Escalate privileges within the domain
* Access sensitive resources such as file shares, databases, and internal applications

You should think of Active Directory as a **large directory that most authenticated users can read from**. Even a basic domain user, with no elevated privileges, can usually enumerate a significant amount of information about the environment.

With a standard domain user account, you can typically discover:

* Domain computers
* Domain users
* Group structures and memberships
* Organisational Units (OUs)
* Default domain and password policies
* Functional domain levels
* Group Policy Objects (GPOs)
* Domain trust relationships
* Access Control Lists (ACLs)

Because of this visibility, it is critical that you understand how Active Directory is structured and administered before attempting to attack it. Knowing how something is built makes it far easier to identify where and why it breaks.

---

## Hierarchical Design

Active Directory is organised as a hierarchy. Understanding this hierarchy is essential, as it directly affects how permissions, policies, and trust relationships behave.

### Forest

* The **forest** sits at the top of the hierarchy
* It represents the main security boundary
* All objects within a forest fall under a shared administrative model

A single forest can contain one or more domains.

### Domain

* A **domain** is a logical container for objects such as:

  * Users
  * Computers
  * Groups
* Domains can contain child or sub-domains
* Each domain includes several default containers and OUs

Domains are where authentication primarily takes place.

### Organisational Units (OUs)

* OUs are used to organise objects within a domain
* They can contain:

  * Users
  * Computers
  * Groups
  * Other OUs
* OUs allow you to:

  * Apply Group Policy selectively
  * Delegate administrative control

This flexibility is powerful but also dangerous when misused.

---

## Example Structure

At a very high level, an Active Directory environment might resemble the following:

```
INLANEFREIGHT.LOCAL/
├── ADMIN.INLANEFREIGHT.LOCAL
│   ├── GPOs
│   └── OU
│       └── EMPLOYEES
│           ├── COMPUTERS
│           │   └── FILE01
│           ├── GROUPS
│           │   └── HQ Staff
│           └── USERS
│               └── barbara.jones
├── CORP.INLANEFREIGHT.LOCAL
└── DEV.INLANEFREIGHT.LOCAL
```

In this example:

* **INLANEFREIGHT.LOCAL** is the root domain
* **ADMIN**, **CORP**, and **DEV** are child domains
* Each domain contains its own objects and policies

<img width="2576" height="1092" alt="ad_forests" src="https://github.com/user-attachments/assets/96d350f5-9862-4a5e-9fa5-37874f08dc43" />


This type of layout is common in medium to large organisations.

---

## Trust Relationships

In real-world environments, you will often see **multiple domains or forests connected by trusts**. This is especially common after mergers or acquisitions, where creating a trust is faster than rebuilding identities from scratch.

<img width="2576" height="1666" alt="ilflog2" src="https://github.com/user-attachments/assets/071930a5-699c-478b-b94d-135889a31205" />


Key points to remember about trusts:

* Trusts define which domains or forests can authenticate to each other
* A trust between root domains does not automatically grant access between all child domains
* Additional trusts may be required for direct communication

Poorly designed trust relationships are a frequent source of security issues. They can unintentionally expose resources or allow attackers to pivot between environments that were assumed to be isolated.
