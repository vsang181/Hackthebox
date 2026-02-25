# Introduction to Password Cracking

Passwords are stored in hashed form to limit the damage of a credential database exposure. A hash function is a one-way mathematical transformation that converts an arbitrary-length input into a fixed-size output, making it computationally infeasible to reverse the output back to the original plaintext. Password cracking is the process of attempting to recover that original input despite this one-way property.

## How Hashing Works

Taking the password `Soccer06!` as an example, the MD5 and SHA-256 digests are generated as follows:

```bash
echo -n Soccer06! | md5sum
40291c1d19ee11a7df8495c4cccefdfa

echo -n Soccer06! | sha256sum
a025dc6fabb09c2b8bfe23b5944635f9b68433ebd9a1a09453dd4fee00766d93
```

The same input will always produce the same output for a given algorithm. This deterministic property is what makes hashes useful for authentication: the stored hash is compared against the hash of the supplied input without ever storing the plaintext. It is also what makes cracking possible, since an attacker who knows the algorithm can hash candidate passwords and compare results against the target hash until a match is found.

## Rainbow Tables

A rainbow table is a large pre-computed mapping of plaintext values to their corresponding hash outputs for a specific algorithm. Because the computation is performed in advance and stored, lookups at runtime are near-instantaneous. A table for MD5 can contain billions of pre-computed entries:

| Password | MD5 Hash |
|---|---|
| `123456` | `e10adc3949ba59abbe56e057f20f883e` |
| `password` | `5f4dcc3b5aa765d61d8327deb882cf99` |
| `iloveyou` | `f25a2fc72690b780b2a14e140ef6a9e0` |
| `rockyou` | `f806fc5a2a0d5ba2471600758452799c` |
| `abc123` | `e99a18c428cb38d5f260853678922e03` |

### Salting

Rainbow tables are defeated by salting. A salt is a random sequence of bytes appended or prepended to a password before hashing. Each user's password receives a unique salt, meaning two users with identical passwords produce entirely different hashes:

```bash
echo -n Th1sIsTh3S@lt_Soccer06! | md5sum
90a10ba83c04e7996bc53373170b5474
```

The salt is not a secret value. It is stored alongside the hash in the database so the system can reconstruct and compare the hash during authentication. Its purpose is uniqueness rather than confidentiality. Even if the correct password exists somewhere in a rainbow table, the salted combination almost certainly does not. Making rainbow tables effective against salted hashes would require recomputing the entire table for every possible salt value: a single-byte salt multiplies the required table size by a factor of 256, turning a 15-billion-entry table into a 3.84-trillion-entry one. Salts of four or more bytes render rainbow table precomputation computationally and storage-wise infeasible.

A related concept is **peppering**, where an additional secret value is incorporated into the hash but stored separately from the database, typically in application configuration. Unlike a salt, a pepper is not stored alongside the hash, so even a full database dump does not give an attacker the complete input required to crack the hash.

## Brute-Force Attacks

A brute-force attack exhausts every possible character combination within a defined keyspace until the target hash is matched. It is the only cracking technique theoretically guaranteed to succeed given sufficient time, since it will eventually reach the correct input regardless of what it is.

In practice, brute-forcing is rarely used in its pure form against passwords longer than eight characters because the time required grows exponentially with length. Cracking speed is heavily dependent on both the algorithm and hardware used. On a standard corporate laptop, Hashcat can test over five million MD5 candidates per second, making a full six-character alphanumeric keyspace exhaustible in minutes. Against DCC2 (Domain Cached Credentials version 2), the same hardware manages only around ten thousand candidates per second because DCC2 is intentionally computationally expensive by design, using multiple PBKDF2 iterations to slow down exactly this kind of attack.

Modern brute-force work is almost always done through **mask attacks** rather than unconstrained enumeration. A mask constrains the character set and structure at each position based on observed password patterns. The mask `?u?l?l?l?d?d?d?s`, for example, targets an uppercase letter, four lowercase letters, three digits, and a special character. This dramatically reduces the search space whilst retaining coverage over the most probable password structures.

## Dictionary Attacks

A dictionary attack substitutes exhaustive character enumeration with a curated list of statistically likely passwords. Because users consistently choose predictable passwords based on common words, names, keyboard patterns, and dates, a well-constructed wordlist recovers the majority of weak passwords in a fraction of the time a brute-force approach would require. This makes dictionary attacks the first choice for assessments operating under time constraints.

Two wordlists are standard resources in penetration testing:

**rockyou.txt** originated from the 2009 breach of RockYou, a social application and widget platform that had stored over 32 million user passwords in plaintext. The leaked passwords were compiled into a single text file that has since become the de facto baseline wordlist for password cracking. Its first twenty entries reflect precisely the weak password patterns most commonly encountered in corporate environments:

```
123456 / 12345 / 123456789 / password / iloveyou
princess / 1234567 / rockyou / 12345678 / abc123
nicole / daniel / babygirl / monkey / lovely
jessica / 654321 / michael / ashley / qwerty
```

In July 2024, a new compilation named RockYou2024 was released, containing nearly 10 billion unique plaintext passwords aggregated from multiple breach datasets. Its existence means that virtually any password that has appeared in a historical breach is already available in an attacker's wordlist.

**SecLists**, maintained by Daniel Miessler on GitHub, is a broader collection of wordlists categorised by use case, including common passwords, usernames, web fuzzing payloads, DNS subdomains, and API endpoint names. Its password-specific lists supplement rockyou.txt with targeted collections for specific languages, industries, and application types, making it the more versatile resource when a target's context is known.

## Technique Selection

The appropriate cracking approach depends on the available time, algorithm, and what is known about the target's password policy:

| Technique | Best Used When |
|---|---|
| **Rainbow table lookup** | Hash is unsalted and the algorithm is weak such as MD5 or NTLM |
| **Dictionary attack** | Time is limited and the password likely follows common patterns |
| **Rule-based attack** | Dictionary alone fails; policy enforces complexity such as capitalisation or appended digits |
| **Mask attack** | Password structure is known or inferred from observed policy requirements |
| **Brute-force** | Password is short (under 8 characters) or all other methods are exhausted |

Rule-based attacks extend dictionary attacks by applying transformation rules to each entry before hashing, for example capitalising the first letter, appending a year, substituting `@` for `a`, or adding a trailing `!`. Hashcat supports hundreds of built-in rules and accepts custom rule files, making rule-based attacks the most practical first approach when the target organisation's password complexity requirements are known.
