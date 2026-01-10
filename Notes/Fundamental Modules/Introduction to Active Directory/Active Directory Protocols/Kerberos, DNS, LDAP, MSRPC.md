# Kerberos, DNS, LDAP, MSRPC

Active Directory does not operate in isolation. It relies on several core protocols to authenticate users, locate resources, and allow systems to communicate with one another. As you study or assess an AD environment, you will repeatedly encounter **Kerberos**, **DNS**, **LDAP**, and **MSRPC**. Understanding how each one fits into the bigger picture is essential.

---

## Kerberos

Kerberos has been the default authentication protocol for domain accounts since Windows 2000. It is an open standard and is designed around **mutual authentication**, meaning both the client and the server verify each other’s identity.

Instead of sending passwords across the network, Kerberos relies on **tickets**, which significantly reduces credential exposure.

### Key concepts you should understand

* Kerberos is **stateless**
* Authentication is **ticket-based**
* User passwords are **never sent over the network**
* Domain Controllers act as a **Key Distribution Centre (KDC)**

### High-level authentication flow

1. When you log in, your password is used to encrypt a timestamp.
2. This encrypted data is sent to the KDC as an authentication request.
3. If the KDC can successfully decrypt the request, it issues a **Ticket Granting Ticket (TGT)**.
4. The TGT is encrypted using the secret key of the `krbtgt` account.
5. You then use the TGT to request a **Ticket Granting Service (TGS)** ticket for a specific service.
6. The TGS is encrypted using the password hash of the target service or computer account.
7. The service validates the ticket and grants access if everything checks out.

This design decouples your credentials from service access. As long as you hold a valid TGT, Kerberos assumes your identity has already been proven.

### Practical notes

* Kerberos uses **TCP and UDP port 88**
* Open port 88 is a strong indicator of a Domain Controller
* Kerberos is a common target for attacks such as:

  * Kerberoasting
  * Ticket forging
  * Delegation abuse

<img width="2576" height="2352" alt="Kerb_auth" src="https://github.com/user-attachments/assets/4227e90e-c885-4542-ad9b-9a0f1edf699a" />

---

## DNS

Active Directory is tightly coupled with **DNS**. Without DNS, AD simply does not function.

DNS is used to:

* Locate Domain Controllers
* Allow DCs to communicate with each other
* Resolve hostnames to IP addresses
* Advertise services via service records

<img width="2576" height="1350" alt="dns_highlevel" src="https://github.com/user-attachments/assets/de8121da-302f-471a-9afd-dce09c84d157" />

### Service Records (SRV)

AD registers **SRV records** in DNS that allow clients to discover services automatically, such as:

* Domain Controllers
* Kerberos services
* LDAP services

When a machine joins a domain, it queries DNS for these records to find a DC.

### Dynamic DNS

Dynamic DNS allows systems to update their DNS records automatically when IP addresses change. Without this, AD environments would be extremely fragile and difficult to manage.

### Ports used by DNS

* **UDP 53** (default)
* **TCP 53** (used when responses are large or UDP fails)

### Common enumeration tasks

You can use DNS to:

* Identify Domain Controllers
* Map internal naming schemes
* Discover hidden hosts and services

Forward lookups resolve names to IPs, while reverse lookups resolve IPs to names. Both are useful during reconnaissance.

---

## LDAP

**Lightweight Directory Access Protocol (LDAP)** is how systems query and interact with Active Directory data.

You should think of LDAP as the language spoken by applications when they need to ask AD questions.

<img width="2576" height="1457" alt="LDAP_auth" src="https://github.com/user-attachments/assets/9ad5b466-2945-498f-8720-71cff5ee3eee" />

### Core characteristics

* Open standard
* Cross-platform
* Latest major version is LDAPv3 (RFC 4511)

### Ports

* **389** – LDAP
* **636** – LDAPS (LDAP over SSL/TLS)

### What LDAP is used for

* Querying users, groups, computers, and policies
* Authenticating credentials
* Retrieving directory attributes
* Managing directory objects

If Kerberos handles authentication tickets, LDAP handles **directory lookups and data access**.

A useful analogy is:

* HTTP is to Apache
* LDAP is to Active Directory

### LDAP authentication methods

LDAP sessions use a **BIND** operation to authenticate.

There are two main approaches:

* **Simple authentication**

  * Anonymous bind
  * Unauthenticated bind
  * Username and password bind
* **SASL authentication**

  * Uses external mechanisms such as Kerberos
  * Separates authentication logic from application logic
  * More secure when configured correctly

### Security note

LDAP traffic is **cleartext by default**. If TLS or LDAPS is not enforced, credentials and queries can be captured on the network. This is a common issue in poorly hardened environments.

---

## MSRPC

**MSRPC** is Microsoft’s implementation of Remote Procedure Call. It allows processes on one system to execute functions on another system.

Windows relies heavily on MSRPC for domain operations, management, and replication.

### Key RPC interfaces used by Active Directory

#### lsarpc

* Interfaces with the Local Security Authority (LSA)
* Manages:

  * Local security policy
  * Audit settings
  * Authentication services
* Commonly used for querying domain security policies

#### netlogon

* Handles authentication between machines and the domain
* Runs continuously in the background
* Essential for domain trust and logon operations

#### samr

* Remote access to the Security Accounts Manager
* Used to manage:

  * Users
  * Groups
  * Computers
* By default, **any authenticated domain user** can query SAMR remotely

This makes samr extremely valuable for enumeration. Tools like BloodHound rely on it to map group memberships and attack paths. Some organisations restrict this via registry settings, but many leave the default behaviour unchanged.

#### drsuapi

* Implements Directory Replication Services
* Used by Domain Controllers to replicate AD data
* Extremely sensitive interface

If abused, drsuapi can be used to:

* Extract the Active Directory database
* Retrieve password hashes for all domain accounts
* Enable attacks such as:

  * DCSync
  * Pass-the-Hash
  * Offline password cracking

---

## Why This Matters

Each of these protocols plays a specific role:

* **Kerberos** controls authentication
* **DNS** enables discovery and communication
* **LDAP** exposes directory data
* **MSRPC** enables management and replication

Misconfigurations in any one of them can undermine the entire security model. As you continue your study or assessments, always consider which protocol is being used, what data it exposes, and how it might be abused. Understanding these interactions is a prerequisite for both effective attacks and meaningful defensive recommendations.
