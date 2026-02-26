# Writing Custom Wordlists and Rules

Effective password cracking in a real engagement is rarely about raw compute power alone. The most impactful factor is the quality and relevance of the wordlist and rule combination being used. A targeted wordlist built around what is known about a specific person or organisation will outperform a generic multi-gigabyte wordlist against that target in almost every case.

## Why Generic Wordlists Fall Short

When a password policy is enforced, users adapt by making the minimum modifications required to satisfy the rules rather than choosing genuinely random passwords. The predictable patterns this produces are well-documented:

| Pattern | Example |
|---|---|
| Capitalise first letter | `Password` |
| Append digits | `Password123` |
| Append year | `Password2022` |
| Append month number | `Password02` |
| Trailing exclamation mark | `Password2022!` |
| Leet substitutions | `P@ssw0rd2022!` |

According to research by WP Engine, most real-world passwords are no longer than ten characters. Users typically select a familiar base word and apply the minimum modifications necessary to pass the policy check. The practical implication is that a wordlist containing personally relevant terms for the target, combined with rules that apply these common transformations, is far more effective than rockyou.txt alone.

## Writing Custom Hashcat Rules

Hashcat rules define transformations applied to each wordlist entry before hashing. Each rule occupies a single line in a plain text file, and multiple functions on the same line are applied left to right in sequence. The full syntax is documented in the [Hashcat rule-based attack wiki](https://hashcat.net/wiki/doku.php?id=rule_based_attack).

### Core Rule Functions

| Function | Description |
|---|---|
| `:` | Do nothing (pass through the word unchanged) |
| `l` | Lowercase all letters |
| `u` | Uppercase all letters |
| `c` | Capitalise the first letter, lowercase the rest |
| `r` | Reverse the entire word |
| `d` | Duplicate the word (`pass` → `passpass`) |
| `sXY` | Replace every instance of character X with character Y |
| `$X` | Append character X to the end |
| `^X` | Prepend character X to the beginning |
| `[` | Remove the first character |
| `]` | Remove the last character |
| `D<n>` | Delete the character at position n |
| `i<n>X` | Insert character X at position n |
| `T<n>` | Toggle the case of the character at position n |

### Building a Custom Rule File

A practical rule file covering common policy-compliant mutations might look like this:

```
:
c
so0
c so0
sa@
c sa@
c sa@ so0
$!
$! c
$! so0
$! sa@
$! c so0
$! c sa@
$! so0 sa@
$! c so0 sa@
```

Applying this rule file to the word `password` produces fifteen variants:

```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list

cat mut_password.list
password
Password
passw0rd
Passw0rd
p@ssword
P@ssword
P@ssw0rd
password!
Password!
passw0rd!
p@ssword!
Passw0rd!
P@ssword!
p@ssw0rd!
P@ssw0rd!
```

The `--stdout` flag instructs Hashcat to output the mutated candidates to standard output rather than attempting to crack any hashes, which is useful for inspecting and saving generated wordlists.

For more complex policies requiring year or number suffixes, additional rules can be appended:

```
$2$0$2$5        # appends 2025
$1$9$9$8        # appends 1998
$! $2$0$2$5     # appends !2025
c $1$9$9$8 $!   # capitalise + append 1998 + !
```

## Generating Target-Specific Wordlists with CeWL

[CeWL](https://github.com/digininja/CeWL) is a Ruby-based web spider that crawls a target URL to a specified depth and extracts unique words from the page content, metadata, and linked pages. The resulting wordlist reflects the vocabulary of the target organisation, making it highly relevant for cracking employee passwords that are based on company terminology, product names, or internal jargon.

```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
wc -l inlane.wordlist
326
```

| Flag | Description |
|---|---|
| `-d <n>` | Depth to spider from the starting URL |
| `-m <n>` | Minimum word length to include |
| `--lowercase` | Normalise all collected words to lowercase |
| `-w <file>` | Write output to a file |
| `--with-numbers` | Include words containing numbers |
| `--ua <agent>` | Set a custom User-Agent string |
| `--auth_type` | Specify HTTP authentication type (basic or digest) |
| `--auth_user / --auth_pass` | Credentials for authenticated pages |
| `-e` | Include email addresses found on the site |

The resulting wordlist should then be combined with a suitable rule file to generate policy-compliant mutations before being used as input to Hashcat or John the Ripper.

## Practical Workflow: Targeted Attack Against a Specific User

The exercise at the end of this section demonstrates the full workflow against a real target (Mark White). The approach below illustrates how OSINT data is translated into a targeted wordlist and rule set.

### Step 1: Build the Base Wordlist from OSINT

From the gathered information, the following terms form the base wordlist:

```
# personal_mark.list
bella          # cat name
maria          # wife name
alex           # son name
baseball       # hobby
nexura         # employer
sanfrancisco   # location
august         # birth month
1998           # birth year
markwhite      # full name
```

### Step 2: Write a Policy-Aware Rule File

Nexura's policy requires at least 12 characters, with uppercase, lowercase, symbol, and digit. The rules below generate candidates that satisfy these requirements from short base words:

```
# mark.rule
c $1 $9 $9 $8 $!           # Capitalise + append 1998 + !
c $2 $0 $2 $5 $!           # Capitalise + append 2025 + !
c $0 $5 $0 $8 $!           # Capitalise + append birth date (0508) + !
c sa@ $1 $9 $9 $8 $!       # Capitalise + a→@ + 1998 + !
c sa@ so0 $1 $9 $9 $8 $!   # Capitalise + a→@ + o→0 + 1998 + !
$! $1 $9 $9 $8 u           # Uppercase all + append 1998!
c $N $e $x $u $r $a $1 $!  # Capitalise + append Nexura1!
```

### Step 3: Generate and Crack

```bash
# Generate the mutated wordlist
hashcat --force personal_mark.list -r mark.rule --stdout | sort -u > mark_mut.list

# Filter out candidates below the 12-character minimum
sed -ri '/^.{,11}$/d' mark_mut.list

# Attempt to crack the MD5 hash
hashcat -a 0 -m 0 97268a8ae45ac7d15c3cea4ce6ea550b mark_mut.list
```

If the targeted wordlist fails, a second pass combining CeWL output against the Nexura website with `best64.rule` is a logical next step. The key principle is to iterate from most-targeted to least-targeted: personal OSINT terms with tight rules first, then company vocabulary with broader rules, then generic wordlists like rockyou.txt with aggressive rule sets like `d3ad0ne.rule` or `dive.rule` as a final fallback.
