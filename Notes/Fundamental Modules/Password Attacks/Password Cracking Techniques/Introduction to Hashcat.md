# Introduction to Hashcat

[Hashcat](https://hashcat.net/) is an open-source, GPU-accelerated password cracking tool supporting Linux, Windows, and macOS. Originally released as proprietary software in 2009, it became open-source in 2015 and has since become the industry standard for offline hash cracking due to its exceptional GPU support, broad hash type coverage, and highly flexible attack system. Its GPU-first architecture means it processes hash comparisons orders of magnitude faster than CPU-based tools like John the Ripper, making it the preferred choice when cracking speed is a priority.

## General Syntax

```bash
hashcat -a <attack_mode> -m <hash_type> <hashes> [wordlist / rule / mask]
```

| Flag | Purpose |
|---|---|
| `-a` | Attack mode (0 = dictionary, 3 = mask, 6/7 = hybrid) |
| `-m` | Hash type ID |
| `-r` | Rule file to apply to wordlist candidates |
| `--show` | Display previously cracked hashes from the potfile |
| `--force` | Ignore warnings and force execution |
| `-o` | Write cracked results to an output file |
| `--session` | Name a session for later restore |
| `--restore` | Resume a previously named session |

## Identifying Hash Types

Each hash algorithm supported by Hashcat is assigned a numeric mode ID. Running `hashcat --help` lists all supported modes. The [Hashcat example hashes wiki](https://hashcat.net/wiki/doku.php?id=example_hashes) provides example outputs for every supported format, which is useful for visually matching an unknown hash against known formats.

For programmatic identification, [hashID](https://github.com/psypanda/hashID) with the `-m` flag returns the corresponding Hashcat mode ID directly:

```bash
hashid -m '$1$FNr44XZC$wQxY6HHLrgrGX0e1195k.1'

[+] MD5 Crypt [Hashcat Mode: 500]
[+] Cisco-IOS(MD5) [Hashcat Mode: 500]
[+] FreeBSD MD5 [Hashcat Mode: 500]
```

Common hash type IDs encountered during assessments:

| Hash Type | Mode ID | Common Source |
|---|---|---|
| MD5 | `0` | Web application databases |
| SHA-1 | `100` | Web application databases |
| SHA-256 | `1400` | Web application databases |
| NTLM | `1000` | Windows SAM / NTDS.dit |
| MD5crypt (`$1$`) | `500` | Linux `/etc/shadow` |
| SHA-512crypt (`$6$`) | `1800` | Linux `/etc/shadow` |
| bcrypt (`$2b$`) | `3200` | Linux `/etc/shadow` |
| DCC2 / mscash2 | `2100` | Windows domain-cached credentials |
| NETNTLMv2 | `5600` | Captured NTLM challenge responses |
| Kerberos 5 TGS | `13100` | Captured Kerberoastable service tickets |
| WPA-PBKDF2-PMKID | `22000` | Captured Wi-Fi handshakes |

## Attack Mode 0: Dictionary Attack

A dictionary attack hashes each entry in a supplied wordlist and compares the result against the target hash. It is the fastest and most commonly successful first approach during an engagement.

```bash
hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 /usr/share/wordlists/rockyou.txt
```

```
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Speed.#1.........: 1706.6 kH/s (0.14ms)
Recovered........: 1/1 (100.00%) Digests
```

### Rules

When a straight dictionary attack fails, rules apply transformations to each wordlist entry before hashing, generating a vastly expanded candidate set without requiring a larger wordlist. Rules execute on the GPU alongside the hash comparisons, making them extremely fast. Hashcat ships with a collection of rule files at `/usr/share/hashcat/rules/`:

| Rule File | Description |
|---|---|
| `best64.rule` | 64 high-yield transformations covering the most common password modifications |
| `rockyou-30000.rule` | 30,000 rules derived from patterns observed in the RockYou dataset |
| `d3ad0ne.rule` | Large rule set with aggressive mutations |
| `dive.rule` | Very large rule set intended for exhaustive rule-based cracking |
| `leetspeak.rule` | Character substitutions such as `a→4`, `e→3`, `i→1`, `o→0` |
| `toggles1`–`toggles5.rule` | Case toggling at varying character positions |
| `T0XlC.rule` | Broad set of leet, number appending, and special character mutations |

Applying `best64.rule` to a wordlist multiplies the candidate count by 64 and covers transformations such as capitalising the first character, reversing the string, appending single digits, and substituting common leet characters:

```bash
hashcat -a 0 -m 0 1b0556a75770563578569ae21392630c /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

```
Status...........: Cracked
Speed.#1.........: 13624.4 kH/s (5.40ms)
Guess.Mod........: Rules (/usr/share/hashcat/rules/best64.rule)
Recovered........: 1/1 (100.00%) Digests
```

Multiple rule files can be stacked by specifying `-r` multiple times. Each file is applied in sequence, compounding the transformations. Chaining `best64.rule` with `leetspeak.rule` for example generates candidates that are both case-modified and leet-substituted.

## Attack Mode 3: Mask Attack

A mask attack is a structured brute-force where the user explicitly defines the character set and positional structure of the candidates rather than testing all possible combinations blindly. This is more efficient than unconstrained brute-force when something is known about the target password's structure, such as its length, casing pattern, or format enforced by an organisation's password policy.

Masks are constructed from a sequence of placeholder symbols, each representing a character set:

| Symbol | Character Set |
|---|---|
| `?l` | `abcdefghijklmnopqrstuvwxyz` |
| `?u` | `ABCDEFGHIJKLMNOPQRSTUVWXYZ` |
| `?d` | `0123456789` |
| `?s` | `!"#$%&'()*+,-./:;<=>?@[\]^_{|}~` and space |
| `?a` | All of `?l`, `?u`, `?d`, and `?s` combined |
| `?h` | Hexadecimal lowercase (`0-9a-f`) |
| `?H` | Hexadecimal uppercase (`0-9A-F`) |
| `?b` | All byte values `0x00`–`0xff` |
| `?1`–`?4` | User-defined custom charsets via `-1`, `-2`, `-3`, `-4` |

Custom charsets can combine built-in sets. For example, `-1 ?d?s` defines `?1` as digits plus special characters, allowing a mask like `?u?a?a?a?a?a?1` to target a seven-character password that starts with an uppercase letter and ends with either a digit or symbol.

To crack a hash where the password is known to be one uppercase letter, four lowercase letters, one digit, and one special character:

```bash
hashcat -a 3 -m 0 1e293d6912d074c0fd15844d803400dd '?u?l?l?l?l?d?s'
```

```
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Guess.Mask.......: ?u?l?l?l?l?d?s  [hashcat](https://hashcat.net/forum/thread-6839.html)
Speed.#1.........:   101.6 MH/s (9.29ms)
Recovered........: 1/1 (100.00%) Digests
```

## Attack Modes Overview

| Mode | Flag | Description |
|---|---|---|
| Dictionary | `-a 0` | Wordlist with optional rules |
| Combination | `-a 1` | Concatenates candidates from two wordlists |
| Mask | `-a 3` | Structured brute-force with explicit character set per position |
| Hybrid (wordlist + mask) | `-a 6` | Appends a mask to each wordlist candidate |
| Hybrid (mask + wordlist) | `-a 7` | Prepends a mask to each wordlist candidate |
| Association | `-a 9` | Associates specific masks or rules to individual hashes |

Hybrid modes 6 and 7 are particularly useful in practice. Mode 6 (`-a 6`) handles the common pattern of a base word followed by a suffix, such as `password2024!`, by combining rockyou.txt candidates with the mask `?d?d?d?d?s`. Mode 7 handles the reverse, where a prefix such as a year precedes a word.
