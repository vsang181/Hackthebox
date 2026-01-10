# Active Directory Objects

When you read about Active Directory, you will constantly see the word **object**. In AD, an object is simply anything that exists inside the directory, such as:

* Users
* Computers
* Groups
* Printers
* Organisational Units (OUs)
* Domain Controllers

In other words, if AD stores it, AD treats it as an object.

# AD Objects

Icons representing Active Directory Objects: Domain, Computer, OU, Groups, User, and Printers.

<img width="2576" height="1440" alt="adobjects" src="https://github.com/user-attachments/assets/c18929a6-b910-4f18-94df-18659ed4b0d5" />

---

## Users

**Users** represent people or accounts that log on and access resources in the domain.

Key points you should remember:

* Users are **leaf objects**, meaning they do not contain other objects
* Users are **security principals**, so they have:

  * A **SID**
  * A **GUID**
* Users have many attributes, including (but not limited to):

  * display name
  * last logon time
  * password last set
  * email address
  * description
  * manager
  * address

Some environments populate only a small subset of user attributes, but the directory supports a very large number of fields. From an attacker’s perspective, user accounts matter because even low-privileged access can enable broad enumeration and provide a starting point for privilege escalation and lateral movement.

---

## Contacts

A **contact** is typically used to store information about someone external to the organisation, such as a vendor or customer.

Characteristics:

* Leaf object
* Not a security principal, so it has:

  * A **GUID**
  * No SID
* Usually contains informational attributes such as:

  * name
  * email address
  * telephone number

---

## Printers

A **printer object** represents a printer that is published in the directory and accessible within the environment.

Characteristics:

* Leaf object
* Not a security principal, so it has:

  * A **GUID**
  * No SID
* Common attributes include:

  * printer name
  * driver information
  * port configuration

---

## Computers

A **computer object** represents a workstation or server that is joined to the domain.

Key points:

* Leaf object
* Security principal, so it has:

  * A **SID**
  * A **GUID**

Computer objects matter during security testing because compromising a host often gives you a powerful execution context (for example, `NT AUTHORITY\SYSTEM`). In many cases, that context allows you to enumerate the domain in a similar way to a standard user, and it can be used to pivot to other targets.

---

## Shared Folders

A **shared folder object** points to a network share hosted on a specific machine.

Characteristics:

* Not a security principal, so it has:

  * A **GUID**
  * No SID
* Common attributes include:

  * share name
  * physical path
  * permissions and access rights

Access to shares varies widely. Depending on configuration, a share may be:

* Publicly accessible to everyone
* Accessible to any authenticated domain user or computer account
* Restricted to specific users or groups

If you do not have explicit access, you will typically be blocked from listing or reading contents.

---

## Groups

A **group** is a container used to manage permissions and access at scale.

Key points:

* Groups are **container objects**, so they can include:

  * users
  * computers
  * other groups
* Groups are **security principals**, so they have:

  * A **SID**
  * A **GUID**

Groups exist to simplify permission management. Instead of granting rights user-by-user, administrators assign permissions to a group and then manage membership.

A common issue you should watch for is **nested groups**, where one group is a member of another. This can create permission chains that administrators did not intend. During assessments, nested membership is frequently used to identify escalation paths. Tools such as BloodHound are often used to visualise and analyse these relationships.

---

## Organisational Units (OUs)

An **Organisational Unit (OU)** is a container that helps administrators organise and delegate control over objects.

You will usually see OUs used for:

* Grouping accounts by department or function
* Delegating administrative tasks without granting full domain admin privileges
* Applying Group Policy to a targeted set of users or computers

Example OU layout:

* Employees (top-level OU)

  * Marketing
  * HR
  * Finance
  * Help Desk

Delegation is one of the main reasons OUs exist. For example:

* Granting password reset rights on the Employees OU would affect everyone
* Granting password reset rights on the Help Desk OU could restrict that capability to only a subset of accounts

Common tasks delegated at the OU level include:

* creating and deleting users
* modifying group membership
* managing Group Policy links
* resetting passwords

OUs also make it easier to apply GPOs to specific subsets of accounts, such as enforcing different policy settings for privileged service accounts.

---

## Domain

A **domain** is the core structure of an Active Directory environment.

A domain:

* Contains objects such as users and computers
* Organises those objects using containers such as groups and OUs
* Maintains its own directory database and policies

Some settings exist by default (for example, password policy), while others are configured to match business needs, such as restricting access to tools or mapping drives during logon.

---

## Domain Controllers

**Domain Controllers (DCs)** provide the core services that make Active Directory function.

In practical terms, DCs:

* process authentication requests
* validate identity within the domain
* enforce security policies
* store directory data for the domain
* control access decisions through roles and permissions

If you compromise a DC, you are typically very close to full domain compromise, because it holds the directory database and is trusted by every domain-joined system.

---

## Sites

A **site** is a logical grouping used to manage replication and traffic flow.

A site typically represents:

* one or more subnets
* connected with high-speed links

Sites exist to make replication between DCs efficient and predictable, especially across multiple physical locations.

---

## Built-in

**Built-in** is a default container created in every domain.

It typically holds:

* predefined groups that exist automatically when the domain is created

These groups often carry powerful rights, so you should always pay attention to membership and delegated permissions involving them.

---

## Foreign Security Principals

A **Foreign Security Principal (FSP)** is a placeholder object created when a security principal from a trusted external forest is referenced inside the current domain.

You will normally see an FSP created when:

* a user, group, or computer from an external forest is added to a group in the current domain

Key characteristics:

* Created automatically
* Stores the **SID** of the external object
* Used by Windows to resolve the name and permissions through the trust relationship
* Stored under the `ForeignSecurityPrincipals` container, for example:

```
cn=ForeignSecurityPrincipals,dc=inlanefreight,dc=local
```
