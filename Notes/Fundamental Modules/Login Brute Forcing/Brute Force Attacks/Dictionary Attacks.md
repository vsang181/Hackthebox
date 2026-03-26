# Dictionary Attacks

A dictionary attack trades theoretical completeness for practical speed. Rather than testing every possible character combination, it tests only combinations that real humans are likely to have chosen, exploiting the fact that most people pick predictable passwords.

***

## Why They Work: Human Psychology

The root vulnerability is not technical, it is behavioural. Users optimise for memorability over security, which means passwords cluster heavily around a small subset of the total possible search space: common words, names, dates, and simple patterns. A wordlist containing one million entries effectively covers a disproportionately large percentage of passwords actually in use, because the distribution of human password choices is not random.

***

## Dictionary Attack vs Brute Force

| Feature | Dictionary Attack | Brute Force |
|--------|-----------------|------------|
| Speed | Fast, limited search space | Slow, exhaustive |
| Targeting | Tailorable to context | No targeting capability |
| Works on complex passwords | No | Yes, given enough time |
| Works on common passwords | Very effectively | Eventually |
| Primary limitation | Misses truly random passwords | Impractical for long passwords |

The practical takeaway for a pentester is to run dictionary attacks first in almost every scenario. If the target uses common passwords, you get results in seconds. If it does not, you have lost very little time before moving to hybrid or brute force approaches.

***

## Key Wordlists and Their Uses

| Wordlist | Size | Best Used For |
|---------|------|-------------|
| `rockyou.txt` | ~14 million | General password attacks, most versatile starting point |
| `2023-200_most_used_passwords.txt` | 200 | Quick check before running larger lists |
| `500-worst-passwords.txt` | 500 | Fast initial sweep, low noise |
| `top-usernames-shortlist.txt` | ~17 | Quick username enumeration |
| `xato-net-10-million-usernames.txt` | 10 million | Thorough username brute forcing |
| `default-passwords.txt` | Varies | Network devices, routers, IoT |

Always start with the smallest relevant list and scale up. Running `rockyou.txt` against a rate-limited endpoint wastes time and risks lockouts when a 200-entry list would have found the password in seconds.

***

## The Dictionary Solver Script Breakdown

```python
import requests

ip = "127.0.0.1"
port = 1234

# Fetch wordlist directly from SecLists
passwords = requests.get(
    "https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master"
    "/Passwords/Common-Credentials/500-worst-passwords.txt"
).text.splitlines()

for password in passwords:
    print(f"Attempted password: {password}")
    response = requests.post(
        f"http://{ip}:{port}/dictionary",
        data={'password': password}
    )
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
```

Three design decisions worth understanding:

- `.splitlines()` cleanly handles both Unix (`\n`) and Windows (`\r\n`) line endings in the downloaded file, avoiding issues where passwords carry a trailing carriage return that would cause every attempt to fail silently
- `data={'password': password}` sends the request as `application/x-www-form-urlencoded`, matching how a real HTML form would submit. Some endpoints require `json={'password': password}` instead, so check the `Content-Type` the server expects if you get unexpected failures
- The `break` on success is essential. Without it the script continues attempting passwords after finding the correct one, generating unnecessary noise in server logs

***

## Building Context-Aware Wordlists

A generic wordlist like `rockyou.txt` is a broad net. A context-aware wordlist built during reconnaissance is a targeted one. Tools exist specifically for this purpose:

```bash
# CeWL: spiders a target website and builds a wordlist from its content
cewl http://target.com -d 2 -m 5 -w target-wordlist.txt

# Manually combining base words with common patterns
# Company: "AcmeCorp", Year: 2024
# Likely passwords: AcmeCorp2024, acmecorp!, Acme2024!, AcmeCorp@2024
```

`CeWL` is particularly effective against company portals, intranets, and admin panels because it harvests the exact vocabulary employees are most likely to use: product names, department names, internal project names, and the company name itself in various forms.

***

## Wordlist Mutation

Most real passwords are not raw dictionary words. They are dictionary words with predictable modifications. This is the gap between a pure dictionary attack and a hybrid attack:

| Base Word | Common Mutations |
|----------|----------------|
| `password` | `Password1`, `p@ssword`, `password123`, `P@ssw0rd` |
| `company` | `Company2024!`, `C0mpany`, `company@1` |
| `summer` | `Summer24`, `summer!`, `Summer2024` |

Tools like `hashcat` with rule files handle this systematically, applying hundreds of mutation rules to every word in a base dictionary to generate a combined list that covers the vast majority of real-world password patterns without requiring a multi-terabyte wordlist.
