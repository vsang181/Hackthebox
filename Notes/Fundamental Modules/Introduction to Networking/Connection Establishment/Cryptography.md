# Cryptography

Cryptography is what makes modern Internet communication possible. Every time you send an email, log in to a website, make an online payment, or connect to a VPN, cryptography is working in the background to keep your data **confidential**, **authentic**, and **protected against manipulation** 

At its core, cryptography uses **mathematical algorithms** to transform readable data (plaintext) into an unreadable form (ciphertext). Without the correct key, this transformed data should be computationally infeasible to recover.

From a security perspective, cryptography is not about being *unbreakable forever*. It is about being **secure enough that breaking it is unrealistic with current resources**.

---

## Encryption Fundamentals

Encryption relies on **keys** and **algorithms**. The same data encrypted with different keys produces different ciphertext. The strength of encryption depends on:

* The algorithm used
* The key length
* The cipher mode
* How keys are generated, stored, and exchanged

In practice, weak cryptography almost always fails due to **implementation or key management**, not mathematics.

---

## Symmetric Encryption

**Symmetric encryption** (also called secret-key encryption) uses **one shared key** for both encryption and decryption.

You and the recipient must both:

* Know the same key
* Protect that key from disclosure

If the key is leaked, lost, or reused improperly, confidentiality is lost immediately.

---

### Where Symmetric Encryption Is Used

Symmetric encryption is fast and efficient, which makes it ideal for:

* Encrypting files and disks
* Securing network traffic after key exchange
* Large volumes of data

Common symmetric algorithms you will encounter:

* **AES (Advanced Encryption Standard)**
* **DES / 3DES** (legacy)

---

### Data Encryption Standard (DES)

DES is a **symmetric block cipher** that encrypts data in 64-bit blocks.

Key facts you should remember:

* Nominal key length: 64 bits
* Effective key length: **56 bits**
* 8 bits are used for parity (checksum)

Because of its short key length, DES is **no longer secure**. Brute-force attacks against DES are trivial with modern hardware.

---

### Triple DES (3DES)

3DES was introduced as an extension of DES.

How it works (simplified):

1. Encrypt with key 1
2. Decrypt with key 2
3. Encrypt with key 3

While more secure than DES, 3DES:

* Is slow
* Still relies on a 56-bit block size
* Is considered deprecated

You should treat 3DES as **legacy-only**.

---

### Advanced Encryption Standard (AES)

AES is the modern standard for symmetric encryption.

Supported key sizes:

* **AES-128**
* **AES-192**
* **AES-256**

Why AES matters:

* Strong security margin
* Efficient performance
* Widely supported in hardware and software

You will find AES everywhere:

* TLS
* IPsec
* SSH
* WLAN (802.11i)
* Disk encryption
* VPNs
* OpenSSL

If encryption is done correctly, **AES is considered secure today**.

---

## Asymmetric Encryption

**Asymmetric encryption** (public-key cryptography) uses **two different keys**:

* A **public key** (shared openly)
* A **private key** (kept secret)

Data encrypted with the public key can only be decrypted with the matching private key.

This solves one of the biggest problems in cryptography: **secure key exchange**.

---

### Why Asymmetric Encryption Is Powerful

Asymmetric encryption allows you to:

* Encrypt data without sharing secrets first
* Authenticate identities using digital signatures
* Establish secure channels over untrusted networks

However, asymmetric encryption is **computationally expensive**, which is why it is rarely used to encrypt large data volumes directly.

---

### Common Asymmetric Algorithms

You will encounter these frequently:

* **RSA** – Based on integer factorisation
* **ECC (Elliptic Curve Cryptography)** – Smaller keys, better performance
* **PGP** – Uses both symmetric and asymmetric encryption

Asymmetric cryptography is widely used in:

* SSL/TLS
* VPNs
* SSH
* PKI
* Cloud platforms
* Digital signatures

---

## Public-Key Encryption in Practice

Asymmetric cryptography is typically used to:

1. Authenticate parties
2. Exchange symmetric keys securely
3. Switch to fast symmetric encryption

This hybrid approach gives you both **security and performance**.

---

## Cipher Modes

Block ciphers (such as AES and DES) encrypt **fixed-size blocks** of data. Cipher modes define **how those blocks are processed**.

Choosing the wrong mode can completely break otherwise strong encryption.

---

### Common Cipher Modes

#### ECB (Electronic Code Book)

* Encrypts blocks independently
* Reveals data patterns
* Vulnerable to statistical analysis

**ECB should never be used.**

---

#### CBC (Cipher Block Chaining)

* Each block depends on the previous block
* Hides patterns effectively
* Requires an initialisation vector (IV)

Commonly used in:

* Disk encryption
* TLS (legacy versions)
* File encryption

---

#### CFB (Cipher Feedback)

* Suitable for streaming data
* Encrypts data as it flows
* Used in network encryption scenarios

---

#### OFB (Output Feedback)

* Stream-oriented
* Resistant to some transmission errors
* Used in SSH and PKCS-based systems

---

#### CTR (Counter Mode)

* Turns block ciphers into stream ciphers
* Fast and parallelisable
* Used in IPsec and disk encryption

---

#### GCM (Galois/Counter Mode)

* Provides **confidentiality and integrity**
* Authenticated encryption
* Highly efficient

GCM is widely used in:

* TLS
* VPNs
* Wireless security

---

## Choosing the Right Mode

You should always prefer:

* **Authenticated encryption** (e.g. AES-GCM)
* Modern, well-reviewed standards
* Strong key lengths

Avoid:

* ECB
* Legacy protocols
* Weak or reused keys

---

## Security Perspective: What Actually Breaks Cryptography

In the real world, cryptography usually fails because:

* Weak key exchange
* Poor randomness
* Reused keys
* Legacy algorithms still enabled
* Misconfigured cipher modes

Rarely because the maths itself is broken.

---

## Key Takeaways

* Cryptography protects confidentiality, integrity, and authenticity
* Symmetric encryption is fast and used for bulk data
* Asymmetric encryption solves key exchange and authentication
* DES and 3DES are obsolete
* AES is the modern symmetric standard
* Cipher modes matter as much as algorithms
* ECB is insecure and should never be used
* GCM provides both encryption and integrity

If you understand cryptography at this level, you will be able to **spot weak designs instantly**, whether you are reviewing code, auditing configurations, or attacking real-world systems.
