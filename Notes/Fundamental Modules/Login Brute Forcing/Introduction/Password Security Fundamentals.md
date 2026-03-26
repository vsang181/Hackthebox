# Password Security Fundamentals

Password strength is the primary variable that determines how long a brute force attack takes. Understanding the maths and the common failure points gives you a clear picture of both the attacker's perspective and what effective defences actually look like.

***

## The Maths of Password Complexity

Each character position in a password multiplies the search space by the size of the character set being used. The relationship is exponential, not linear, which is why even small increases in length or character variety have dramatic effects:

| Password Profile | Character Set Size | Combinations |
|----------------|-------------------|-------------|
| 6 chars, lowercase only | 26 | ~300 million |
| 8 chars, lowercase only | 26 | ~200 billion |
| 8 chars, mixed case + digits | 62 | ~218 trillion |
| 12 chars, mixed case + digits + symbols | 95 | ~540 quadrillion |

This is why NIST guidelines prioritise length above all else. Going from 8 to 12 characters with the same character set increases the search space by a factor of roughly 1.7 million, while adding symbols to an 8-character password increases it by a much smaller factor. Length wins.

***

## The Four Properties of a Strong Password

NIST's current guidance organises password strength around four properties:

- Length: minimum 12 characters, with longer passphrases actively encouraged over short complex strings
- Complexity: mixing character classes expands the per-position possibilities, though NIST has de-emphasised mandatory complexity rules in recent guidelines because they tend to produce predictable patterns like `P@ssw0rd1`
- Uniqueness: each account requires its own credential so a breach at one service does not cascade across others
- Randomness: no dictionary words, personal information, or keyboard patterns, since these are the first entries tested in any wordlist-based attack

***

## Common Weaknesses Attackers Exploit

From a pentesting standpoint, these are the weaknesses worth actively testing for in order of how commonly they produce results:

1. Default credentials left unchanged on network devices, printers, and cameras
2. Credential reuse across services (addressed by credential stuffing attacks)
3. Predictable patterns: `Company2024!`, `Welcome1`, `Summer2025`
4. Personal information: names, birthdates, pet names pulled from LinkedIn or social media
5. Short passwords under 8 characters, common in legacy systems with old policies
6. Simple substitutions: `@` for `a`, `0` for `o`, `3` for `e`, which all major wordlists already account for

***

## Default Credentials as Low-Hanging Fruit

Default passwords represent the easiest win in any assessment. The table in the module illustrates how many enterprise-class devices ship with credentials as simple as `admin:admin` or `admin:1234`. Attackers maintain curated default credential lists specifically for this reason, and tools like Hydra can test an entire default credential database against a target in seconds.

The more overlooked problem is default usernames. Even when an administrator changes the password, keeping `admin` or `root` as the username gives an attacker half the credential for free. SecLists' `top-usernames-shortlist.txt` contains the most common default usernames, and running it as your username wordlist before attempting password fuzzing is always worth doing first.

***

## Password Policies: The Security-Usability Tension

Overly strict password policies frequently backfire. When users are forced to change passwords every 30 or 60 days and cannot reuse any of the last ten, the typical response is incremental passwords: `Password1!`, `Password2!`, `Password3!`. This satisfies the policy mechanically while providing almost no actual security improvement because attackers account for this pattern in hybrid wordlist attacks.

NIST's current SP 800-63B guidance explicitly recommends against mandatory periodic rotation unless there is evidence of compromise, and against complexity rules that produce predictable patterns. The more effective controls are:

- Minimum length of at least 12 characters
- Checking new passwords against known breach databases (Have I Been Pwned API)
- Blocking common passwords outright
- Implementing MFA as a second factor rather than relying solely on password strength

***

## Pentester Takeaways

From an offensive standpoint, understanding password security translates directly into attack sequencing:

1. Try default credentials first: zero cost, often succeeds on network devices and internal services
2. Check for credential reuse using breach databases if usernames are already known
3. Run targeted dictionary attacks using context-aware wordlists (company name, service name, year, common patterns)
4. Add hybrid mutations (appending digits and symbols to dictionary words) before attempting full brute force
5. Reserve pure brute force exclusively for offline hash cracking where there is no lockout risk

The depth of a target's password policy also tells you how aggressive to be. A system with no lockout policy and no rate limiting can sustain a high-thread dictionary attack. A system that locks after five attempts requires careful password spraying with long delays between rounds.
