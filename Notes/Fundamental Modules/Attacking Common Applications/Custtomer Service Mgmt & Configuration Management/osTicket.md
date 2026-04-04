# osTicket: Discovery, Enumeration, and Attack Paths
[osTicket](https://osticket.com/) is an open-source PHP/MySQL support ticketing system used widely by companies, universities, and government bodies to handle user support requests. While it has a relatively clean CVE history compared to other common applications, it represents a different category of risk: sensitive operational data flowing through it makes it valuable even when no vulnerability exists. Support ticketing systems should never be overlooked during assessments.

***
## Identification
osTicket does not expose itself through Nmap service detection since the scan will only return the underlying web server (Apache or IIS). Instead, identify it through:

- A cookie named `OSTSESSID` set on the session
- The footer text "powered by osTicket" or "Support Ticket System"
- The login path at `/scp/login.php` for staff access
- The ticket submission page at `/open.php`

The page title and branding are usually unchanged from the default, making visual identification straightforward during an EyeWitness run.

***
## CVE-2020-24881: SSRF
[CVE-2020-24881](https://nvd.nist.gov/vuln/detail/CVE-2020-24881) is a Server-Side Request Forgery vulnerability affecting osTicket versions before 1.14.3, carrying a CVSS score of 9.8.  An attacker can attach a malicious file reference to a ticket, causing the server to make outbound requests to internal hosts or perform internal port scanning.  This can be leveraged to probe services inside the network that are not exposed externally, or in some configurations to reach internal metadata endpoints such as those on cloud-hosted instances. 
***
## Email Address Harvesting via Ticket Creation
This attack requires no vulnerability in osTicket itself and works against any correctly functioning installation. Many osTicket deployments assign a temporary internal email address to each new ticket so users can check its status by replying to it.

The chain works as follows:

1. Browse to the ticket submission page at `/open.php` and open a new ticket under any pretext
2. After submission, the system provides a ticket reference and an internal email address in the format `<ticketid>@companydomain.com`
3. Any email sent to that address appears inside the ticket thread when you log in to check the ticket
4. Use this valid company email address to register for other internal or external services that require company email verification, such as Slack, Mattermost, GitLab, or Bitbucket

This exact technique was the basis for the HTB machine [Delivery](https://0xdf.gitlab.io/2021/05/22/htb-delivery.html) and is documented in a widely cited [bug bounty writeup](https://medium.com/intigriti/how-i-hacked-hundreds-of-companies-through-their-helpdesk-b7680ddc2d4c) that demonstrated it against hundreds of real companies.

***
## Credential Reuse and Sensitive Data in Tickets
When credentials are found through external OSINT sources such as [Dehashed](https://dehashed.com/) or breach data, test them against the osTicket staff login at `/scp/login.php`. The login form accepts both username and email address, so test both formats:

```
jclayton / JulieC8765!
kevin@inlanefreight.local / Fish1ng_s3ason!
```

If access is gained to a staff or agent account, the ticket queue is often a goldmine. Common sensitive data found in ticket threads includes:

- Helpdesk agents sending passwords directly to users in ticket replies
- Standard onboarding passwords shared across multiple accounts
- Internal hostnames, IP addresses, and application names mentioned in troubleshooting threads
- Personal employee information that aids further social engineering

The osTicket address book is also worth exporting entirely, as it may contain staff email addresses and internal usernames not found elsewhere.

***
## Password Spraying After Ticket Data
If a standard onboarding or reset password is discovered in ticket correspondence, it becomes a candidate for password spraying across other exposed services. The workflow is:

1. Identify the standard password from ticket data
2. Build an employee username list using [linkedin2username](https://github.com/initstring/linkedin2username), which scrapes LinkedIn and generates common username formats: 

```bash
python3 linkedin2username.py -c inlanefreight -n inlanefreight.local
```

3. Spray the discovered password against exposed services such as VPN portals, OWA, or Citrix

This is particularly effective when the domain password policy does not require a change on first login, which is common in organisations with lenient Active Directory policies.

***
## Defensive Takeaways
These attack paths are preventable with a few straightforward controls:

- Limit support portals to internal access only where possible, and avoid external exposure
- Train helpdesk staff never to send credentials or sensitive data through ticket threads
- Enforce MFA on all external portals without exception
- Require users to change their password on first login and enforce password expiry
- Prevent corporate email addresses from being used to sign up for third-party services through policy and monitoring
- Export and review the osTicket address book periodically to audit who has access
