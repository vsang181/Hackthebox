# Password Policies

Password policies define the rules for how passwords must be created, managed, and maintained across an organisation. They serve a dual purpose: enforcing a minimum security baseline and guiding users toward consistently strong credential choices. A policy without enforcement is ineffective, and enforcement without clear communication leaves users struggling to comply.

## Industry Standards

Rather than building a policy from scratch, organisations typically align with one or more established security standards. Each takes a distinct stance on requirements like expiration and complexity: [blog.qualys](https://blog.qualys.com/product-tech/2026/02/11/qualys-etm-detect-pass-the-hash-pass-the-ticket-attacks)

| Standard | Key Focus |
|---|---|
| [NIST SP800-63B](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-63b.pdf) | Favours length over complexity, discourages forced rotation, mandates breach-list checking |
| [CIS Password Policy Guide](https://www.cisecurity.org/insights/white-papers/cis-password-policy-guide) | Practical controls for enterprise environments, maps to CIS Benchmarks |
| [PCI DSS](https://www.pcisecuritystandards.org/document_library?category=pcidss&document=pci_dss) | Compliance-driven, minimum 12-character passwords, MFA requirements for administrative access |

Compliance with any of these standards does not guarantee security on its own — it establishes a baseline. The organisation's actual risk posture determines how far beyond baseline controls should extend.

## The Password Expiration Debate

One of the most contested policy decisions is forced expiration. The traditional "change your password every 90 days" approach has been largely deprecated by modern guidance. The reason is behavioural: when forced to rotate passwords frequently, users predictably increment a number or append a character rather than creating a genuinely new credential. As covered in the Password Mutations section, attackers are well aware of these patterns — `Inlanefreight01!` becoming `Inlanefreight02!` at expiry provides no meaningful security improvement. Current industry consensus recommends disabling routine expiration and requiring a password change only when a specific compromise is confirmed or suspected.

## Building a Meaningful Policy

A technically compliant password can still be weak. Consider these two outcomes from the same policy requiring eight characters, mixed case, a number, and a special character:

- `password123!` — meets requirements, trivially guessable
- `CjDC2x[U` — meets requirements, genuinely strong

Minimum complexity rules alone do not prevent weak passwords. A well-constructed policy should additionally include a blocklist of predictable choices. Common categories to blocklist:

- The company name and common internal terminology
- Months and seasons
- Variations on "welcome", "password", "login", and "changeme"
- Commonly leaked passwords such as `123456`, `qwerty`, and `letmein`
- Employee names and usernames

## Enforcing the Policy

Definition and enforcement are separate concerns that must both be addressed. In Active Directory environments, an [Active Directory Password Policy GPO](https://activedirectorypro.com/how-to-configure-a-domain-password-policy/) handles technical enforcement — minimum length, complexity requirements, lockout thresholds, and history depth can all be configured centrally. For finer control, including custom blocklists, tools like Microsoft Entra ID Password Protection or third-party solutions extend what native Group Policy can enforce.

Technical enforcement must be paired with clear communication. Users who do not understand why a policy exists will find ways around it — choosing the minimum compliant password rather than a genuinely secure one. Onboarding training, visible policy documentation, and descriptive error messages (as Mark received) all contribute to actual compliance rather than surface-level compliance.

## Creating Passwords That Are Both Strong and Memorable

Randomly generated passwords like `CjDC2x[U` produced by tools such as [1Password's generator](https://1password.com/password-generator/) are cryptographically strong but difficult to commit to memory. A practical alternative is the passphrase approach — combining ordinary words into a long, memorable phrase and adding complexity through special characters:

```
()The name of my dog is Popy!
This is my secure password for HTB
```

Length is the single biggest contributor to crack resistance. A 25-character passphrase made of common words is far harder to crack by brute force than an 8-character random string, and considerably easier to remember. One important caveat: avoid passphrases that draw on personal information attackable through OSINT — pet names, family names, or location references that could be found on social media weaken an otherwise strong construction.

As the number of unique passwords grows, remembering them individually becomes impractical. The next section addresses this directly by exploring how a password manager can generate, store, and organise credentials securely at scale.
