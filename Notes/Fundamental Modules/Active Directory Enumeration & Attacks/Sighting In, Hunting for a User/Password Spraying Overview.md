## Password Spraying Overview

Password spraying is an attack where a single common password is attempted against a large list of usernames or email addresses to gain access to an exposed service. Usernames are typically gathered during the OSINT phase or initial enumeration. Unlike brute forcing, which hammers one account with many passwords, spraying spreads attempts across many accounts with very few passwords, making it harder to trigger lockout policies.

***

## Real-World Scenarios

### Scenario 1

Standard checks such as SMB NULL sessions and LDAP anonymous binds returned nothing useful. [Kerbrute](https://github.com/ropnop/kerbrute) was used to build a valid username list by combining the [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) wordlist with data scraped from LinkedIn. Spraying the resulting list with the password `Welcome1` produced two hits on low-privileged accounts. Those accounts provided enough domain access to run [BloodHound](https://github.com/BloodHoundAD/BloodHound) and identify attack paths that led to full domain compromise.

### Scenario 2

Common username lists and LinkedIn scraping produced no results in the second assessment. Searching Google for PDFs published by the organisation revealed that four documents contained an internal username format of `F9L8` in their Author metadata field, a randomly generated GUID using capital letters and numbers only. The following Bash script generated all 1,679,616 possible combinations:

```bash
#!/bin/bash

for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}
    do echo $x;
done
```

Running this full list through [Kerbrute](https://github.com/ropnop/kerbrute) enumerated every single account in the domain. Starting with a complete account list significantly improves spray success rates compared to the typical 40-60% coverage achieved with a list like `jsmith.txt`. Valid passwords were recovered for several accounts, and the chain was eventually extended using [Resource-Based Constrained Delegation (RBCD)](https://posts.specterops.io/another-word-on-delegation-10bdbe3cd94a) and the [Shadow Credentials](https://web.archive.org/web/20250703081952/https://www.fortalicesolutions.com/posts/shadow-credentials-workstation-takeover-edition) attack to achieve full domain control.

***

## How a Password Spray Works

Each spray attempt uses one password across all target accounts before moving to the next password. A deliberate delay is inserted between rounds to avoid hitting the lockout threshold.

| Round | Username | Password |
|-------|----------|----------|
| 1 | bob.smith@inlanefreight.local | Welcome1 |
| 1 | john.doe@inlanefreight.local | Welcome1 |
| 1 | jane.doe@inlanefreight.local | Welcome1 |
| DELAY | | |
| 2 | bob.smith@inlanefreight.local | Passw0rd |
| 2 | john.doe@inlanefreight.local | Passw0rd |
| 2 | jane.doe@inlanefreight.local | Passw0rd |
| DELAY | | |
| 3 | bob.smith@inlanefreight.local | Winter2022 |
| 3 | john.doe@inlanefreight.local | Winter2022 |
| 3 | jane.doe@inlanefreight.local | Winter2022 |

***

## Key Considerations

Careless spraying can lock out hundreds of production accounts, which causes real operational harm to a client environment. Knowing the domain password policy before starting significantly reduces that risk, and internal access often makes it possible to retrieve the policy directly.

Common policy configurations to be aware of:

- Five bad attempts before lockout with a 30-minute auto-unlock threshold is a typical setup
- Some organisations require manual administrator intervention to unlock accounts
- If the policy is unknown, waiting a few hours between rounds is a safe default

Practical guidelines for conducting a spray responsibly:

- Always try to retrieve the password policy before spraying during an internal assessment
- If no policy is obtainable, treat it as a single "hail mary" attempt using one weak or common password only after other foothold options are exhausted
- Ask the client to clarify the policy if the type of assessment allows for it
- If you already hold a foothold or a test account, enumerate the policy through that access before spraying
