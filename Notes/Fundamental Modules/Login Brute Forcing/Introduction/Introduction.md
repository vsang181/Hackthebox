# Brute Forcing Fundamentals

Brute forcing is a trial-and-error attack method that systematically tests credential combinations until a valid one is found. It is one of the oldest attack techniques in existence, but remains highly relevant because weak password practices are still widespread across organisations of all sizes.

***

## What Determines Success or Failure

Three variables govern whether a brute force attempt succeeds within a practical timeframe:

- Password complexity: each additional character and character class multiplies the search space exponentially. An 8-character lowercase-only password has 26^8 (roughly 200 billion) combinations, while adding uppercase, digits, and symbols pushes that into the quadrillions
- Computational power: modern GPUs can test billions of combinations per second against offline hashes, but online attacks against web forms are throttled by network latency and server response time
- Defensive controls: lockout policies, CAPTCHAs, rate limiting, and MFA can reduce online attack viability to near zero regardless of hardware capability

***

## Attack Types and When to Use Each

The eight methods covered in the module each suit different circumstances:

| Method | Core Mechanic | When It Fits |
|--------|-------------|-------------|
| Simple Brute Force | Every possible combination | No password intelligence available |
| Dictionary Attack | Pre-compiled wordlists | Target likely uses common passwords |
| Hybrid Attack | Dictionary words plus character mutations | Target may have modified a common word |
| Credential Stuffing | Reusing leaked credential pairs | Large breach database available |
| Password Spraying | Few passwords against many accounts | Lockout policies in place |
| Rainbow Table | Pre-computed hash lookups | Many hashes to crack offline |
| Reverse Brute Force | One password against many usernames | Suspected password reuse |
| Distributed Brute Force | Workload split across many machines | Extremely complex target with no lockout |

Password spraying and credential stuffing are the most operationally relevant in real-world penetration tests because they respect lockout thresholds while still producing results. Pure brute force is almost exclusively useful against offline targets such as captured password hashes or encrypted files.

***

## Brute Forcing in the Penetration Testing Context

Brute forcing sits toward the later stages of a penetration test, typically employed after softer methods have been exhausted. The three scenarios that warrant it are straightforward: all other access vectors have failed, the target's password policy is visibly weak (short minimums, no complexity requirements, no expiry), or a specific high-value account such as a domain admin or service account needs to be targeted directly.

An important distinction for authorised testing is the difference between online and offline attacks. Online attacks hit a live system and are inherently noisy, slow, and risky if lockout policies exist. Offline attacks work against captured hashes from a database dump or a cracked handshake and carry none of those constraints since no live system is being queried.

***

## Defences That Matter Most

Understanding the attack types directly informs which controls to recommend when assessing a target:

- Account lockout and progressive delays defeat simple brute force and targeted dictionary attacks
- MFA defeats credential stuffing entirely since the leaked password alone is insufficient
- Password managers and unique-per-site passwords defeat credential stuffing and reverse brute force
- Salted hashes defeat rainbow table attacks since the pre-computed tables become useless
- Monitoring for distributed login attempts across IPs defeats distributed brute force and spraying when alerts are properly tuned

The module's coverage of Hydra and Medusa will put these concepts into practice against SSH, FTP, and web forms, where each protocol introduces its own constraints around speed, threading, and how to identify a successful login attempt.
