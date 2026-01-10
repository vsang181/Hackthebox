# Active Directory Functionality

This section focuses on how Active Directory actually functions behind the scenes. To work effectively with AD, you need to understand how responsibility is distributed, how compatibility is enforced, and how trust boundaries are created and abused.

---

## FSMO Roles

Active Directory uses **Flexible Single Master Operation (FSMO)** roles to prevent conflicts in multi-domain-controller environments. Instead of allowing every Domain Controller (DC) to perform every operation, specific critical tasks are assigned to specific roles.

There are **five FSMO roles**, split across forest-wide and domain-wide scopes.

### Forest-wide FSMO Roles

These roles exist once per forest.

* **Schema Master**

  * Controls read and write access to the Active Directory schema
  * Determines which object classes and attributes can exist in AD
  * Required when extending the schema (for example, Exchange installation)

* **Domain Naming Master**

  * Ensures domain names are unique within the forest
  * Required when adding or removing domains

### Domain-wide FSMO Roles

These roles exist once per domain.

* **Relative ID (RID) Master**

  * Allocates RID pools to DCs
  * Prevents duplicate SIDs from being issued
  * Each object SID is built from:

    * Domain SID
    * RID assigned to the object

* **PDC Emulator**

  * Acts as the authoritative DC for:

    * Authentication
    * Password changes
    * Account lockouts
    * Group Policy updates
  * Maintains time synchronisation for the domain

* **Infrastructure Master**

  * Resolves references between objects in different domains
  * Translates SIDs, GUIDs, and Distinguished Names
  * If this role fails, ACLs may display unresolved SIDs instead of names

FSMO roles are initially assigned when a forest or domain is created, but administrators can transfer them if needed. Problems with FSMO roles often lead to authentication failures, policy issues, or inconsistent directory behaviour.

---

## Domain and Forest Functional Levels

**Functional levels** define which Active Directory features are available and which Windows Server versions are allowed to act as Domain Controllers.

You should think of functional levels as compatibility gates. Raising them enables newer features but removes support for older systems.

### Domain Functional Levels (High-Level Overview)

Each domain functional level inherits features from the level below it.

* **Windows 2000 native**

  * Universal groups
  * Group nesting and conversion
  * SID history

* **Windows Server 2003**

  * Constrained delegation
  * Selective authentication
  * Introduction of `lastLogonTimestamp`

* **Windows Server 2008**

  * DFS-R support
  * Kerberos AES encryption
  * Fine-grained password policies

* **Windows Server 2008 R2**

  * Managed Service Accounts
  * Authentication mechanism assurance

* **Windows Server 2012**

  * Claims-based authentication
  * Kerberos armouring

* **Windows Server 2012 R2**

  * Protected Users group
  * Authentication policies and silos

* **Windows Server 2016**

  * Enhanced credential protection
  * Smart card enforcement improvements

No new domain functional level was introduced with Windows Server 2019. However:

* Windows Server 2008 functional level is the **minimum** for Server 2019 DCs
* SYSVOL **must** use DFS-R (not FRS)

### Forest Functional Levels

Forest functional levels introduce features that apply across all domains in the forest.

Key milestones include:

* **Windows Server 2003**

  * Forest trusts
  * Domain renaming
  * Read-Only Domain Controllers (RODCs)

* **Windows Server 2008 R2**

  * Active Directory Recycle Bin

* **Windows Server 2016**

  * Privileged Access Management (PAM) via Microsoft Identity Manager

Raising functional levels is a one-way operation and must be planned carefully.

---

## Trusts

A **trust** allows authentication to flow between domains or forests. Trusts are what make multi-domain and multi-forest environments usable, but they are also one of the most common sources of security issues.

A trust links the authentication systems of two domains.

<img width="2576" height="1666" alt="trusts-diagram" src="https://github.com/user-attachments/assets/142b9692-bb20-4c33-9b3b-36dd8c007e71" />

---

## Trust Types

You will commonly encounter the following trust types:

* **Parent-child**

  * Automatically created within a forest
  * Two-way and transitive

* **Cross-link**

  * Created between child domains
  * Used to optimise authentication paths

* **External**

  * Non-transitive trust between separate forests
  * Uses SID filtering by default

* **Tree-root**

  * Created when adding a new tree to a forest
  * Two-way and transitive

* **Forest**

  * Trust between forest root domains
  * Transitive across all domains in both forests

---

## Trust Direction and Scope

Trusts differ in both **direction** and **scope**.

### Transitive vs Non-transitive

* **Transitive**

  * Trust extends beyond the immediate domain
  * Applies to child domains as well

* **Non-transitive**

  * Trust applies only to the explicitly trusted domain

### One-way vs Two-way

* **Two-way (bidirectional)**

  * Users in both domains can access resources in each other

* **One-way**

  * Access flows in one direction only
  * The direction of trust is opposite to the direction of access

---

## Why Trusts Matter for Security

Trust relationships are frequently misconfigured or forgotten after initial setup.

Common risk scenarios include:

* Old trusts left behind after mergers or acquisitions
* Bidirectional trusts created for convenience
* Lack of review or monitoring of trusted domains

These issues can create unintended attack paths. It is not uncommon to compromise one domain, perform attacks such as Kerberoasting, and discover credentials that grant administrative access in another, more sensitive domain.

As you move forward, always treat trusts as potential pivot points. Understanding how they work is critical for both attacking and defending Active Directory environments.
