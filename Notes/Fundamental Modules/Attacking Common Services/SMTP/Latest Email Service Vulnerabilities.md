# Latest Email Service Vulnerabilities

[CVE-2020-7247](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-7247) is a critical RCE vulnerability in OpenSMTPD up to version 6.6.2, exploitable without authentication and present in the codebase since 2018 before being publicly disclosed in 2020. It affected major Linux and BSD distributions including Debian, Fedora, and FreeBSD.

## The Vulnerability

The flaw lives in the function responsible for recording the sender's email address during the SMTP session. The code does not properly sanitise the input, allowing an attacker to inject a semicolon (`;`) to break out of the expected input context and append arbitrary shell commands directly into the data that OpenSMTPD processes. There is a 64-character limit on the injected command, which constrains complexity but does not prevent effective payloads -- a short reverse shell or a wget command to pull down a second-stage payload fits comfortably within that limit. Full technical details are documented on the [OpenWall oss-security mailing list](https://www.openwall.com/lists/oss-security/2020/01/28/3), and a working exploit is available on [Exploit-DB](https://www.exploit-db.com/exploits/47984).

## Mapping to the Concept of Attacks

The attack runs through two cycles.

### Cycle 1 -- Initiating the SMTP Interaction

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker initiates a connection to the SMTP service and begins composing an email, either manually or via an automated script | Source |
| 2 | The service accepts the email and begins processing the provided fields including sender, recipient, and message body | Process |
| 3 | OpenSMTPD listens on port 25, which requires root privileges to bind -- the service therefore runs with elevated privileges | Privileges |
| 4 | The input data is forwarded to an internal OpenSMTPD process for further handling | Destination |

### Cycle 2 -- Triggering Remote Code Execution

| Step | What Happens | Category |
|---|---|---|
| 5 | The full input, particularly the sender field containing the injected semicolon and shell command, becomes the source for the next processing step | Source |
| 6 | The parsing function reads the sender field and the semicolon character triggers a code path that executes the appended string as a shell command rather than treating it as data | Process |
| 7 | Because OpenSMTPD runs with root privileges, all child processes it spawns -- including the injected command -- inherit those same elevated privileges | Privileges |
| 8 | The attacker's machine becomes the destination, receiving a reverse shell or command output sent back over the network | Destination |

## Why This Pattern Keeps Appearing

OpenSMTPD, SMBGhost, BlueKeep, and the xp_dirtree MSSQL technique all demonstrate the same structural reality: services that require elevated privileges to function -- binding to low-numbered ports, managing kernel resources, interacting with hardware -- automatically elevate the impact of any vulnerability found within them. A flaw in a service running as an unprivileged user is a problem. The same flaw in a service running as root or LocalSystem is a full system compromise.

## Recommended Practice

The module notes several Hack The Box machines that demonstrate real email attack chains worth working through:

- [SneakyMailer](https://0xdf.gitlab.io/2020/11/28/htb-sneakymailer.html) covers phishing and IMAP inbox enumeration using Netcat
- [Reel](https://0xdf.gitlab.io/2018/11/10/htb-reel.html) covers SMTP user brute-forcing and phishing via malicious RTF files
- Rabbit covers brute-forcing Outlook Web Access (OWA) and macro-based phishing

The site [ippsec.rocks](https://ippsec.rocks/) is useful for finding which HTB machines cover any specific attack technique -- searching a term like "SMTP" or "open relay" returns a list of relevant machines, making it an efficient way to find targeted practice rather than working through boxes at random.
