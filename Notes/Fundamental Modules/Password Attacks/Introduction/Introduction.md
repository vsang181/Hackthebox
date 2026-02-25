# Introduction

Passwords remain the dominant authentication mechanism in enterprise environments despite being the authentication factor most vulnerable to attack. A 2025 study of over 19 billion leaked credentials found that only 6% of exposed passwords are unique, with 94% being reused or duplicated across accounts, making credential stuffing and password spraying disproportionately effective against real-world targets.  Understanding authentication as a complete system, not just its password component, is essential for identifying where it can be bypassed during an assessment. 
## Authentication Factors

Authentication is the process of validating identity by presenting one or more factors to a validation mechanism. NIST defines multi-factor authentication as requiring two or more of the following categories: 
| Factor | Category | Examples |
|---|---|---|
| **Something you know** | Knowledge | Password, PIN, passphrase, security question |
| **Something you have** | Possession | Smart card, hardware token, TOTP authenticator app, trusted mobile device |
| **Something you are** | Inherence | Fingerprint, facial recognition, retina scan, voice pattern |
| **Somewhere you are** | Location | Geolocation, IP address range, office network boundary |

Security policy determines which factors are required based on the sensitivity of the resource being accessed.  A low-sensitivity internal wiki may require only a username and password. Access to medical records terminals, by contrast, commonly requires a smart card (possession) combined with a PIN (knowledge), with some organisations adding a biometric third factor for high-assurance environments. The more factors required, the higher the authentication assurance, but also the greater the operational friction placed on users. 

From an attacker's perspective, each factor represents a separate attack surface. Knowledge factors can be guessed, cracked offline, or phished. Possession factors can be stolen, cloned, or intercepted via SIM swapping and real-time phishing proxies. Inherence factors are the most difficult to compromise remotely but are not immune to replay attacks when their outputs are transmitted as static data. Location factors are often the weakest control, as IP address and geolocation spoofing are trivial.

## The Password Problem

Passwords persist as the primary authentication factor not because they are the most secure option, but because they offer the best balance of cost, usability, and compatibility across systems. Complex authentication deployments, hardware token infrastructure, and biometric readers all introduce implementation cost, user training burden, and support overhead. A username and password requires no physical hardware, works across every platform, and is understood by every user. This usability advantage explains their persistence despite their well-documented weaknesses.

The statistical picture of real-world password usage remains poor. From a 2025 Cybernews analysis of 19 billion leaked credentials: 

- `123456` appeared in 338 million exposed passwords and has been among the most common passwords every year since at least 2011
- `password` appeared in 56 million and `admin` in 53 million entries
- 94% of passwords in the dataset were reused or duplicated across multiple accounts
- Only 6% of analysed passwords were unique

A Google and Harris Poll survey found that 66% of Americans reuse the same password across multiple platforms, meaning that a single recovered credential from one compromised service has a statistically significant probability of granting access to other accounts belonging to the same user.  The same survey found that only 45% of Americans would change their passwords following a confirmed breach, meaning more than half of users whose credentials appear in a breach database continue using the exposed password. 

The practical implication for penetration testing is clear: a single successful password recovery, whether through cracking a captured hash, finding credentials in a leaked database, or observing plaintext credentials in transit, frequently unlocks access well beyond the originally compromised service. Password reuse across VPN portals, email, domain accounts, and internal applications is one of the most reliable paths from initial access to broad network compromise.

## Keyspace and Cracking Feasibility

The theoretical strength of a password is determined by its keyspace, the total number of possible combinations given its character set and length. A standard 8-character password using only uppercase letters and digits has a keyspace of \(36^8 = 208{,}827{,}064{,}576\) possible values, approximately 208 billion combinations. Modern GPU-accelerated cracking tools such as [Hashcat](https://hashcat.net/hashcat/) can test NTLM hashes at rates exceeding 100 billion hashes per second on consumer hardware, meaning this entire keyspace is exhausted in under two seconds against an NTLM hash.

In practice, passwords are rarely random. Users choose passwords based on meaningful words, names, dates, keyboard patterns, and predictable substitutions such as `@` for `a` or `3` for `e`.  A Cybernews 2025 analysis found that 27% of unique passwords consisted only of lowercase letters and digits, and nearly 20% mixed letter cases and numbers but included no special characters.  Rule-based cracking against realistic wordlists exploits this predictability and routinely recovers passwords that would be theoretically resistant to pure brute-force given their length. 
## Relevance to This Module

This module addresses the full lifecycle of password attacks encountered during an assessment:

- **How passwords are stored** across Windows, Linux, and application platforms, including the specific hash formats and storage locations relevant to each
- **How to retrieve stored credentials** from memory, registry hives, shadow files, and database exports
- **Offline cracking methods** including dictionary attacks, rule-based mutations, mask attacks, and hybrid approaches
- **Pass-the-Hash and related techniques** for authenticating with hashes that cannot be cracked in a feasible timeframe
- **Identifying weak and default credentials** on network services, web applications, and devices through targeted testing

Accounts worth checking for breach exposure can be verified at [HaveIBeenPwned](https://haveibeenpwned.com/), which indexes email addresses against reported breach databases and is a useful OSINT resource during the reconnaissance phase of an assessment.
