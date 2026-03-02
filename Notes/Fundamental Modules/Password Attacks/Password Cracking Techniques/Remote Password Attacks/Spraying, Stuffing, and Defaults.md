# Spraying, Stuffing, and Defaults

Password spraying, credential stuffing, and default credential abuse are three distinct but related techniques that exploit the same underlying weakness: poor password hygiene. Together they account for a significant proportion of real-world initial access. According to Verizon's 2025 DBIR, compromised credentials were the initial access vector in 22% of all breaches reviewed, and infostealer analysis revealed that in the median case, only 49% of a user's passwords across different services were distinct from one another — meaning the other half were reused.

## Password Spraying

[Password spraying](https://owasp.org/www-community/attacks/Password_Spraying_Attack) tests a single password against a large number of accounts rather than exhausting many passwords against a single target. The key advantage is that it avoids account lockout: most policies lock an account after a threshold of failed attempts against that specific account, but because each account only receives one attempt, the lockout threshold is never reached.

Spraying is particularly effective in two scenarios:

- **Newly provisioned accounts** where an IT team sets a standard temporary password such as `Welcome1!`, `ChangeMe123!`, or `Company2025!` and some users never update it
- **Seasonal or predictable patterns** such as `Summer2025!` or `Jan2026!` that users adopt to satisfy complexity requirements at mandatory password reset intervals

[NetExec](https://github.com/Pennyw0rth/NetExec) is the primary tool for spraying across Active Directory environments, supporting subnet ranges to cover an entire network segment in a single command:

```bash
netexec smb 10.100.38.0/24 -u usernames.list -p 'ChangeMe123!'
```

The `--no-bruteforce` flag is important when the intent is to test each username against its paired password rather than all combinations. For spraying a single password across all users, the above syntax without `--no-bruteforce` is correct, as the single-password constraint already limits the attempt count per account to one.

[Kerbrute](https://github.com/ropnop/kerbrute) provides an alternative approach specifically for Active Directory, using Kerberos pre-authentication to validate credentials without generating LDAP or NTLM authentication event logs, making it significantly stealthier than SMB or WinRM-based spraying:

```bash
kerbrute passwordspray -d corp.local usernames.list 'ChangeMe123!'
```

For web applications, [Burp Suite](https://portswigger.net/burp) Intruder with a Pitchfork or Cluster Bomb attack configuration allows fine-grained control over request timing, making it possible to spray a login form while respecting per-account attempt limits.

### Operational Considerations

Spraying carries real operational risk if the lockout policy is not known in advance. Before launching any spray, the password policy should be enumerated. NetExec can retrieve it from SMB:

```bash
netexec smb 10.100.38.0/24 --pass-pol
```

Key policy fields to note are the observation window (the period over which failed attempts are counted) and the lockout threshold (the number of failures that trigger a lockout). Spraying one password per observation window keeps the attempt count per account below the threshold.

## Credential Stuffing

[Credential stuffing](https://owasp.org/www-community/attacks/Credential_stuffing) replays known username and password pairs, typically sourced from public breach databases, against a target service. Unlike spraying, the attacker already has a specific password for each username and is testing whether it was reused on the target platform. A 2025 analysis of SSO provider logs found that credential stuffing accounted for a median of 19% of all daily authentication attempts across monitored organisations, reaching as high as 25% at enterprise scale.

[Hydra](https://github.com/vanhauser-thc/thc-hydra) supports credential stuffing directly with the `-C` flag, which accepts a colon-separated `username:password` file rather than separate user and password lists:

```bash
hydra -C user_pass.list ssh://10.100.38.23
```

The input format for `-C` is one pair per line:

```
jsmith:Summer2023!
m.jones:Welcome1
admin:Password123
```

Breach databases in this format are readily available through sources such as [HaveIBeenPwned](https://haveibeenpwned.com/), public leak repositories, and dark web markets. The `rockyou2024.txt` compilation released in mid-2024 contains approximately 10 billion unique plaintext credentials aggregated from hundreds of historical breaches, making it a comprehensive source of known credential pairs for stuffing attacks.

The distinction between spraying and stuffing from a detection perspective matters during an engagement. Stuffing attempts generate one failed or successful login per unique account, which looks like distributed low-volume activity across many users — harder to correlate through simple per-account lockout logic but detectable through behavioural analytics that look at cross-account failure spikes.

## Default Credentials

Default credentials are factory-set username and password combinations shipped with network devices, applications, databases, IoT hardware, and management interfaces. Despite being widely documented and trivial to find, they remain one of the most consistently successful attack vectors. An IBM study found that 86% of routers have had their default passwords left unchanged. In 2025, CISA issued guidance directly urging manufacturers to eliminate default passwords entirely following high-profile incidents where critical infrastructure was accessed using credentials as simple as `1111`.

Default credentials are dangerous not only because they are known but because they are predictable at scale. An attacker who compromises one device with `admin:admin` in an environment where hundreds of identical devices were deployed simultaneously can often access all of them with the same credential.

### Tools and Sources

The [DefaultCreds-cheat-sheet](https://github.com/ihebski/DefaultCreds-cheat-sheet) is a community-maintained database of known default credentials for thousands of products, installable as a Python command-line tool:

```bash
pip3 install defaultcreds-cheat-sheet

creds search linksys

+---------------+---------------+------------+
| Product       |    username   |  password  |
+---------------+---------------+------------+
| linksys       |    <blank>    |  <blank>   |
| linksys       |    <blank>    |   admin    |
| linksys       | Administrator |   admin    |
| linksys       |     admin     |   admin    |
| linksys (ssh) |     admin     |  password  |
| linksys (ssh) |      root     |   admin    |
+---------------+---------------+------------+
```

Other useful resources include:
- **[CIRT.net](https://cirt.net/passwords)** — searchable database of over 2,000 products and their default credentials
- **Vendor documentation** — installation and quick-start guides almost always list the default credential alongside the setup instructions
- **[Shodan](https://www.shodan.io/)** — internet-facing devices running services with default credentials can be found through service banner searches

### Common Router Defaults

Routers are a particularly high-value target in internal engagements, as compromise of a router provides network-level access and the ability to intercept or redirect traffic. Devices used in internal test labs or staging environments are especially likely to retain default credentials:

| Brand | Default IP | Default Username | Default Password |
|---|---|---|---|
| 3Com | `192.168.1.1` | `admin` | `Admin` |
| Belkin | `192.168.2.1` | `admin` | `admin` |
| D-Link | `192.168.0.1` | `admin` | `Admin` |
| Linksys | `192.168.1.1` | `admin` | `Admin` |
| Netgear | `192.168.0.1` | `admin` | `password` |
| Digicom | `192.168.1.254` | `admin` | `Michelangelo` |

### Workflow for Default Credential Testing

The recommended workflow during an assessment is:

1. **Enumerate services** — identify the application, firmware version, or device model from banners, web interfaces, or Nmap service detection
2. **Query default credential databases** — use `creds search`, CIRT.net, or vendor documentation to retrieve known defaults for identified products
3. **Build a credential list** in `username:password` format
4. **Test with Hydra using `-C`** or manually against the service's authentication interface, applying appropriate rate limits to avoid service disruption

```bash
# Build a colon-separated list from defaults
echo "admin:admin" >> defaults.list
echo "admin:password" >> defaults.list
echo "root:admin" >> defaults.list

# Test against an SSH service
hydra -C defaults.list ssh://10.100.38.23

# Test against a web login form
hydra -C defaults.list http-post-form "10.100.38.23/login:user=^USER^&pass=^PASS^:Invalid credentials"
```
