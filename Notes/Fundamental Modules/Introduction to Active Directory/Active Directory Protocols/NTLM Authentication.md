# NTLM Authentication

In addition to Kerberos and LDAP, Active Directory environments still rely on several legacy authentication mechanisms. These mechanisms are widely supported for backward compatibility, but they are also frequently abused. As you work through AD assessments, you must clearly understand the difference between **hash types** and **authentication protocols**, as they are not the same thing.

Hashes describe how a password is stored.
Protocols describe how authentication is performed using those hashes.

Active Directory commonly involves the following:

* LM
* NTLM
* NTLMv1
* NTLMv2

Kerberos is generally preferred where possible, but NTLM-based authentication is still common due to legacy systems, misconfigurations, and application requirements.

---

## Hash and Protocol Overview

At a high level:

* NTLM-based protocols use **symmetric cryptography**
* They **do not support mutual authentication**
* They rely on challenge-response mechanisms
* The Domain Controller acts as the trusted authority

Kerberos improves on these limitations by introducing mutual authentication and ticket-based access.

---

## LAN Manager (LM)

**LM hashes** are the oldest password storage mechanism in Windows environments.

Important characteristics:

* Introduced in the late 1980s
* Stored in:

  * SAM (local systems)
  * NTDS.DIT (Domain Controllers)
* Disabled by default since Windows Vista / Server 2008
* Still encountered in legacy environments

### Security weaknesses you should remember

* Passwords are limited to **14 characters**
* Passwords are **not case-sensitive**
* Passwords are converted to **uppercase**
* Passwords are split into **two 7-character chunks**
* Each chunk is hashed independently

This design massively reduces the keyspace and makes LM hashes extremely fast to crack. If a password is 7 characters or fewer, half of the LM hash is always identical, making it trivial to identify.

LM hashes can and should be disabled via Group Policy.

---

## NT Hash (NTLM)

**NTLM** uses the NT hash, which is far stronger than LM but still problematic.

Key points:

* NT hash is calculated as:

  * MD4(UTF-16-LE(password))
* Stored in:

  * SAM
  * NTDS.DIT
* Supports the full Unicode character set

<img width="2576" height="1316" alt="ntlm_auth" src="https://github.com/user-attachments/assets/f3fae4ae-9380-4a6a-99c0-754a93d5cb22" />

NTLM authentication uses a **challenge-response** model with three messages:

1. Negotiate
2. Challenge
3. Authenticate

Despite being stronger than LM, NT hashes can still be:

* Cracked offline using GPUs
* Used directly in **pass-the-hash** attacks

This means an attacker does not need the plaintext password to authenticate, only the hash itself.

### NTLM hash structure example

An NTLM entry typically contains:

* Username
* RID
* LM hash (often disabled or empty)
* NT hash

The NT hash is the part attackers care about most, as it can be cracked or replayed.

---

## NTLMv1 (Net-NTLMv1)

**NTLMv1** builds on NTLM by using a challenge-response exchange during network authentication.

How it works at a high level:

* Server sends an 8-byte random challenge
* Client responds with a computed value based on the challenge and the hash
* The response is 24 bytes long

Key limitations:

* Uses both LM and NT hashes
* Easier to crack offline than newer variants
* Cannot be used for pass-the-hash attacks

Because of these weaknesses, NTLMv1 is considered insecure and should be disabled wherever possible.

---

## NTLMv2 (Net-NTLMv2)

**NTLMv2** was introduced to address many of the flaws in NTLMv1 and has been the default for many years.

Improvements include:

* Stronger cryptographic construction
* Use of HMAC-MD5
* Inclusion of:

  * Client challenge
  * Timestamp
  * Domain information

NTLMv2 sends two responses:

* One based on a fixed-length challenge
* One based on variable-length data including time and domain name

This makes NTLMv2 significantly harder to crack than NTLMv1, though still not immune to offline attacks under certain conditions.

---

## Important NTLM Limitations

You should keep the following in mind at all times:

* NTLM does **not** support mutual authentication
* NTLM hashes are **not salted**
* NTLM is vulnerable to:

  * Pass-the-hash
  * Relay attacks
  * Offline cracking (depending on password strength)

These weaknesses are the reason modern environments should prioritise Kerberos wherever possible.

---

## Domain Cached Credentials (MSCache2)

Domain Cached Credentials exist to solve a specific problem: what happens when a domain-joined machine cannot contact a Domain Controller.

### How it works

* The system caches hashes for the **last ten domain users** who logged in
* Stored in:

  * `HKEY_LOCAL_MACHINE\SECURITY\Cache`
* Used when:

  * Network connectivity is unavailable
  * Domain authentication cannot be performed

### Security characteristics

* Hashes **cannot** be used for pass-the-hash
* Extremely slow to crack
* Designed to resist GPU-based attacks

Even with powerful hardware, cracking these hashes is usually impractical unless the password is extremely weak or highly targeted.

As an assessor, you may extract these hashes after gaining local administrator access, but you must carefully evaluate whether cracking attempts are worth the time and effort.
