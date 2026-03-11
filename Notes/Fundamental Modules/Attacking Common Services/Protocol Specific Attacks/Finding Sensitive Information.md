# Finding Sensitive Information

Finding sensitive information during an assessment is fundamentally a detective process. Every piece of data collected, no matter how small it appears in isolation, can be the link that connects one service to another and ultimately leads to the objective.

## Why Every Detail Matters

The scenario above illustrates this perfectly. An empty file named `johnsmith` on an anonymous FTP share appears worthless at first glance. It contains no credentials, no configuration data, and no direct path to exploitation. But that filename is a username, and usernames are valid intelligence. Testing it against adjacent services revealed a working email login, which then exposed database credentials inside an old email, which then enabled command execution on the MSSQL server. Each step was only possible because the previous one was not dismissed as irrelevant.

This chain -- anonymous FTP access to email access to database access to RCE -- did not require a single software exploit. Every step was built on information gathered from a misconfigured or carelessly managed service.

## Categories of Sensitive Information

When enumerating any service, the goal is to identify anything that could advance the attack or be of value to a real adversary:

- Usernames and email addresses -- useful for credential attacks, password spraying, and OSINT
- Passwords and password hashes -- direct access or offline cracking opportunities
- DNS records and IP addresses -- network topology and discovery of additional targets
- Source code -- logic flaws, hardcoded credentials, internal API keys
- Configuration files -- service accounts, connection strings, infrastructure details
- Personally identifiable information (PII) -- relevant for scoping data breach impact during reporting

## Services Covered in This Module

The three service categories most likely to hold actionable sensitive information in a typical enterprise environment are:

- **File shares** (SMB, FTP, NFS) -- configuration files, scripts with hardcoded credentials, backup archives, and internal documentation are frequently found on poorly managed shares
- **Email** -- password reset threads, credentials sent between staff, internal system notifications, and onboarding documents containing default credentials
- **Databases** -- application credential tables, customer data, internal tooling configurations, and in some cases direct OS-level command execution capability

## The Two Requirements for Finding Sensitive Information

Regardless of the service being targeted, the same two conditions determine whether sensitive information will be found and recognised:

**Understanding how the service works.** Without knowing that MSSQL supports `xp_cmdshell`, or that FTP allows anonymous login, or that IMAP lets you search mailbox content programmatically, the attack surface of each service is invisible. Technical knowledge of the service determines what questions to ask and what enumeration steps to take.

**Knowing what to look for.** This requires understanding the target -- its business model, its processes, and what data it handles. A healthcare company's sensitive information looks different from a financial institution's. An internal IT team's shared drive will contain different material than a customer-facing web server. The more familiar you are with the target's context, the better you can prioritise what you find and recognise what is genuinely valuable versus what is noise.

Both elements reinforce each other. Technical knowledge tells you where to look, and contextual knowledge tells you what matters when you find it. Together they form the foundation of effective service enumeration throughout the rest of this module.
