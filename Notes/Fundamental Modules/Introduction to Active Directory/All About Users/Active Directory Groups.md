# Active Directory Groups

After user accounts, **groups** are one of the most important objects in Active Directory. Groups allow you to collect users, computers, and other objects together and assign access or permissions in bulk. From both an administrative and an offensive perspective, groups are extremely powerful.

As you work through AD environments, you will quickly notice that group-based permissions are not always obvious. Rights granted through group membership may be indirect, inherited, or buried several layers deep. This makes groups a prime target during penetration tests and a common source of excessive or unintended privileges in real environments.

Most domains contain a mixture of:

* Built-in groups created automatically by Active Directory
* Custom groups created to support business or operational needs

Over time, the number of groups can grow rapidly. Without regular review, this sprawl often leads to poor visibility, privilege creep, and security risk. Understanding how group types and scopes work is essential if you want to reason about access properly.

---

## Groups vs Organisational Units

A common point of confusion is the difference between **groups** and **Organisational Units (OUs)**.

<img width="1018" height="510" alt="group-options2" src="https://github.com/user-attachments/assets/03f261d7-ef91-49f8-9e13-1a50c235c342" />

You should think of them as serving different purposes:

* **Groups**

  * Primarily used to grant access to resources
  * Control permissions and rights
  * Influence what members can do

* **OUs**

  * Used to organise objects
  * Used to apply Group Policy
  * Used to delegate administrative tasks

An OU can delegate administrative actions such as password resets without granting broader rights that might be inherited through group membership. Groups, on the other hand, directly affect access control.

---

## Why groups exist

Groups exist to simplify management.

If you need to give 50 users access to a file share, adding permissions to each user individually is slow, error-prone, and difficult to audit. Instead, you assign permissions to a group and manage membership.

This approach allows you to:

* Grant or revoke access by changing group membership
* Keep permissions consistent
* Audit access more easily

If a user no longer needs access, you remove them from the group and leave everything else untouched.

---

## Group type and scope

Every Active Directory group has two defining properties:

* **Type** – what the group is used for
* **Scope** – where the group can be used

When you create a group, both must be selected.

---

## Group Types

There are two group types in Active Directory.

### Security Groups

Security groups are used to assign permissions and rights.

Key characteristics:

* Can be granted access to resources
* Members inherit permissions assigned to the group
* Used heavily throughout AD environments

These are the groups you will most often care about during security assessments.

### Distribution Groups

Distribution groups are used by email systems.

Key characteristics:

* Used for email distribution lists
* Cannot be used to assign permissions
* Common in environments with Exchange or similar systems

From a security perspective, distribution groups are usually less interesting.

---

## Group Scopes

Group scope determines **where** a group can be used and **what it can contain**.

There are three group scopes.

---

### Domain Local Groups

Domain local groups are used to control access to resources within a single domain.

Key points:

* Permissions apply only within the domain where the group exists
* Can contain users and groups from other domains
* Can be nested inside other domain local groups
* Cannot be nested inside global groups

These groups are commonly used to control access to specific servers or resources.

---

### Global Groups

Global groups are designed to represent collections of users from a single domain.

Key points:

* Can only contain objects from the same domain
* Can be used to grant access to resources in other domains
* Can be nested inside:

  * Other global groups
  * Domain local groups

Global groups are often used to represent job roles or departments.

---

### Universal Groups

Universal groups are used in multi-domain environments.

Key points:

* Can contain users and groups from any domain in the forest
* Can be granted permissions anywhere in the forest
* Stored in the Global Catalog

Because universal groups are stored in the Global Catalog:

* Any membership change triggers forest-wide replication
* Excessive changes can cause unnecessary replication traffic

Best practice is to place **global groups** inside universal groups, rather than individual users. Global group membership changes replicate only within a domain, reducing overhead.

---

## Changing group scope

Group scopes can be changed, but there are important restrictions:

* A global group can be converted to universal only if it is not a member of another global group
* A domain local group can be converted to universal only if it does not contain other domain local groups
* A universal group can always be converted to domain local
* A universal group can only be converted to global if it does not contain other universal groups

These limitations matter when restructuring environments or cleaning up legacy design.

---

## Built-in vs custom groups

Active Directory creates many **built-in groups** automatically when a domain is created. These groups are intended for specific administrative purposes and often carry significant privileges.

Important points:

* Many built-in groups are domain local
* Some do not allow group nesting
* Some accept only user accounts as members

For example:

* **Domain Admins** is a global group and can only contain users from its own domain
* **Administrators** is a domain local group and can contain users from other trusted domains

Custom groups are extremely common. Organisations create them to reflect business structure, access needs, or application requirements. Some products also create groups automatically. For example, installing Exchange introduces several highly privileged groups. If these groups are not reviewed and managed carefully, they can become escalation paths.

---

## Nested group membership

Nested group membership is one of the most important concepts to understand in Active Directory.

A user may gain privileges not because of direct membership, but because:

* Their group is a member of another group
* That group is nested multiple layers deep

This can result in users having permissions they were never explicitly intended to have.

During assessments, nested group membership is frequently abused. It is also one of the hardest issues for administrators to reason about manually. Tools that visualise group relationships are extremely useful for identifying these indirect paths.

In practice, a user might:

* Be a member of Group A
* Group A is a member of Group B
* Group B has permissions over a sensitive resource

Even though the user is not directly in Group B, they still inherit those rights.

---

## Important group attributes

Groups have many attributes, but a few are particularly useful during enumeration and analysis.

Key attributes include:

* **cn**

  * The group’s common name

* **member**

  * Objects that belong to the group

* **memberOf**

  * Groups that this group is a member of

* **groupType**

  * Encodes both group type and scope

* **objectSid**

  * The SID used during access control decisions

Understanding these attributes helps you trace permission inheritance and identify where access is coming from.
