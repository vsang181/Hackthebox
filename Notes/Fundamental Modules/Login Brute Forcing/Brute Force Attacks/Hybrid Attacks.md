# Hybrid Attacks

Hybrid attacks combine dictionary and brute force techniques into a single, more effective approach. They are designed specifically to exploit the predictable patterns humans adopt when forced to change passwords, bridging the gap between the speed of dictionary attacks and the coverage of brute force.

***

## Why Hybrid Attacks Work

When password change policies exist without adequate user education, people follow a mental shortcut: take the existing password and make the smallest possible modification that satisfies the policy. This produces patterns that are neither truly random nor in any static wordlist:

- `Summer2023` becomes `Summer2024` or `Summer2023!`
- `Password1` becomes `Password2` or `Password1!`
- `Company` becomes `Company2024` or `Company@1`

A pure dictionary attack misses these because the modified versions are not in any wordlist. A pure brute force attack wastes enormous time on combinations that no human would ever choose. A hybrid attack covers both by taking dictionary words and systematically applying mutations.

***

## Filtering Wordlists to Match Password Policies

One of the most practical hybrid techniques is reducing a large wordlist to only entries that satisfy the target's known password policy. This eliminates obviously invalid candidates before any requests are sent, cutting the search space dramatically.

The four-stage grep pipeline from the module achieves this efficiently:

```bash
# Stage 1: Minimum 8 characters
grep -E '^.{8,}$' darkweb2017_top-10000.txt > filtered-minlength.txt

# Stage 2: At least one uppercase letter
grep -E '[A-Z]' filtered-minlength.txt > filtered-uppercase.txt

# Stage 3: At least one lowercase letter
grep -E '[a-z]' filtered-uppercase.txt > filtered-lowercase.txt

# Stage 4: At least one digit
grep -E '[0-9]' filtered-lowercase.txt > filtered-final.txt

# Check how many candidates remain
wc -l filtered-final.txt
```

The result is 89 candidates from a starting list of 10,000. That is a 99.1% reduction in the search space before a single authentication attempt is made. Each regex stage is chained on the output of the previous one, so every surviving entry satisfies all policy requirements simultaneously.

This same technique can be compressed into a single command using piped grep:

```bash
grep -E '^.{8,}$' darkweb2017_top-10000.txt | \
grep -E '[A-Z]' | \
grep -E '[a-z]' | \
grep -E '[0-9]' > filtered-final.txt
```

***

## Regex Reference for Policy Filtering

| Policy Requirement | Regex Pattern |
|------------------|--------------|
| Minimum 8 characters | `^.{8,}$` |
| At least one uppercase | `[A-Z]` |
| At least one lowercase | `[a-z]` |
| At least one digit | `[0-9]` |
| At least one special character | `[^a-zA-Z0-9]` |
| Maximum 16 characters | `^.{1,16}$` |
| No spaces | `^[^ ]+$` |

These can be combined into a single grep using lookahead assertions if needed, though chaining is easier to read and debug during an assessment.

***

## Hashcat Rules for Mutation-Based Hybrid Attacks

For offline hash cracking, hashcat rule files are the standard tool for applying mutations to a base wordlist. Rules can append, prepend, substitute, capitalise, and apply dozens of other transformations:

```bash
# Apply the best64 rule set to rockyou.txt against captured hashes
hashcat -a 0 -m 0 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Use the d3ad0ne rule for more aggressive mutations
hashcat -a 0 -m 0 hashes.txt rockyou.txt -r /usr/share/hashcat/rules/d3ad0ne.rule
```

Common mutation rules from `best64.rule` include appending `1`, `123`, `!`, and `2024`, capitalising the first letter, replacing `a` with `@`, `o` with `0`, and `e` with `3`. These cover the vast majority of real-world password modification patterns.

***

## Credential Stuffing

Credential stuffing is a distinct but related technique that treats breached credential pairs as a wordlist rather than generating new combinations. The attack logic is:

1. Obtain leaked username:password pairs from breach databases
2. Identify services the victims are likely to use
3. Automate login attempts against those services using the exact leaked pairs
4. Gain access to accounts where passwords were reused unchanged

The success rate of credential stuffing attacks is consistently high, reported at 0.1% to 2% in industry studies, because password reuse remains extremely common despite widespread awareness. At the scale of modern breaches (hundreds of millions of leaked credentials), even a 0.1% success rate against a target service translates to thousands of compromised accounts.

***

## Defences That Break Each Attack

| Attack Type | Most Effective Defence |
|------------|----------------------|
| Hybrid / pattern-based | Passphrase policy, password manager adoption |
| Credential stuffing | MFA, breached password checking on login |
| All online attacks | Rate limiting, CAPTCHA, account lockout |
| Offline hash cracking | Strong hashing algorithms (bcrypt, Argon2), high work factors |

Breached password checking deserves more attention than it typically receives. Services like Have I Been Pwned provide an API that lets applications reject passwords that appear in known breach datasets at the point of registration or password change, cutting off credential stuffing before it starts.
