# Authentication Protocols

Authentication protocols exist to answer a very simple but critical question:

**Who are you, really?**

In networking and security, this question must be answered **securely, repeatedly, and reliably**. Without authentication protocols, any system would struggle to distinguish legitimate users and devices from attackers, making unauthorised access trivial and widespread.

Authentication protocols also play a key role in **secure information exchange**. Many of them combine identity verification with encryption, integrity checking, and protection against replay or man-in-the-middle attacks. Understanding how these protocols work, and where they fail, is essential from both a defensive and offensive perspective 

---

## Why Authentication Protocols Matter

In real environments, authentication protocols are used to:

* Verify user identities
* Authenticate devices and services
* Establish trust between systems
* Secure remote access
* Protect sensitive data in transit

If authentication is weak, everything built on top of it becomes unreliable.

---

## Commonly Used Authentication Protocols

Below is a practical overview of authentication protocols you will encounter frequently. This is not about memorising acronyms, but understanding **where and why** each one is used.

| Protocol     | Purpose                                                                                                            |
| ------------ | ------------------------------------------------------------------------------------------------------------------ |
| **Kerberos** | Ticket-based authentication using a Key Distribution Centre (KDC), commonly used in Active Directory environments. |
| **SRP**      | Password-based authentication protocol designed to resist eavesdropping and man-in-the-middle attacks.             |
| **SSL**      | Legacy cryptographic protocol for securing communication (now deprecated).                                         |
| **TLS**      | Modern successor to SSL, providing secure communication over the Internet.                                         |
| **OAuth**    | Authorisation framework that allows third-party access without sharing passwords.                                  |
| **OpenID**   | Decentralised authentication system enabling a single identity across multiple services.                           |
| **SAML**     | XML-based standard for exchanging authentication and authorisation data between trusted parties.                   |
| **2FA**      | Uses two independent factors to verify identity.                                                                   |
| **FIDO**     | Open standards for strong, phishing-resistant authentication.                                                      |
| **PKI**      | Public Key Infrastructure for managing certificates and digital identities.                                        |
| **SSO**      | Single sign-on mechanism allowing one login for multiple services.                                                 |
| **MFA**      | Multi-factor authentication using two or more verification factors.                                                |
| **PAP**      | Sends passwords in clear text (insecure and obsolete).                                                             |
| **CHAP**     | Uses a challenge-response mechanism instead of sending passwords directly.                                         |
| **EAP**      | Framework supporting multiple authentication methods, widely used in network access control.                       |
| **SSH**      | Secure protocol for remote command execution and authentication.                                                   |
| **HTTPS**    | Secure HTTP using TLS for encrypted web communication.                                                             |
| **LEAP**     | Cisco wireless authentication protocol (deprecated and insecure).                                                  |
| **PEAP**     | Secure EAP-based tunnelling protocol using TLS.                                                                    |

---

## Wireless and Network Authentication (LEAP vs PEAP)

Protocols such as **LEAP** and **PEAP** are commonly used for authenticating wireless clients and remote users.

### LEAP (Lightweight EAP)

* Developed by Cisco
* Uses shared secrets
* Relies on RC4 encryption
* Vulnerable to dictionary attacks
* Does not protect MSCHAPv2 hashes

LEAP should be considered **insecure** and deprecated.

---

### PEAP (Protected EAP)

* Encapsulates authentication inside a TLS tunnel
* Uses server-side certificates
* Encrypts MSCHAPv2 hashes
* Supports stronger encryption (AES, 3DES)
* Widely deployed in enterprise environments

PEAP is significantly more secure than LEAP but is still vulnerable to certain attacks if poorly configured. Modern environments increasingly prefer **EAP-TLS**.

---

## Authentication Over Encrypted Channels

For most wired and remote-access scenarios, authentication is performed over encrypted protocols such as:

* **SSH**
* **HTTPS**

These protocols provide:

* Encryption (confidentiality)
* Integrity protection
* Server authentication using certificates
* Resistance to passive eavesdropping

They also integrate well with **PKI**, allowing clients to validate server identities and reducing the risk of man-in-the-middle attacks.

---

## Security Perspective: What You Should Look For

When assessing authentication in a real environment, always ask yourself:

* Is authentication encrypted?
* Are legacy protocols still enabled?
* Are shared secrets reused?
* Is certificate validation enforced?
* Are weak ciphers or hashes allowed?
* Is MFA properly implemented or just cosmetic?

Weak authentication almost always leads to **full compromise**, not partial access.

---

## Key Takeaways

* Authentication protocols verify identity and establish trust
* Encryption alone is meaningless without strong authentication
* Legacy protocols like PAP and LEAP are insecure
* TLS-based authentication is the modern baseline
* Wireless authentication is a common weak point
* Misconfigured authentication is a high-impact attack vector

If you understand authentication protocols deeply, you will quickly recognise **where trust is misplaced**, **where credentials can be stolen**, and **where access can be escalated**.
