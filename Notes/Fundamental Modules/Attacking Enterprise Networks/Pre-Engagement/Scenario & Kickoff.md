## Scenario and Kickoff

### Engagement Overview

Client: Inlanefreight
Testing Company: Acme Security, Ltd.
Assessment Type: Full-scope External Penetration Test, transitioning to Internal if a foothold is gained.

The client wants to identify as many vulnerabilities as possible from the perspective of an anonymous internet user. Evasive testing is not required. If the [DMZ](https://www.cloudflare.com/learning/network-layer/what-is-a-dmz/) is breached and internal network access is achieved, the engagement extends to full internal testing up to and including [Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) domain compromise.

No credentials have been provided for web applications, VPN, or Active Directory.

### Scope

**External Testing**
- `10.129.x.x` (external-facing target host)
- `*.inlanefreight.local` (all subdomains)
- `INLANEFREIGHT.LOCAL` (Active Directory domain)

**Internal Testing**
- `172.16.8.0/23`
- `172.16.9.0/23`

The client has not provided a list of specific subdomains or live hosts. Discovery is required to map attacker visibility against both the external and internal network.

### Out of Scope

- Phishing or social engineering against any Inlanefreight employees or customers
- Physical attacks against Inlanefreight facilities
- Destructive actions or Denial of Service (DoS) testing
- Any environment modifications without written consent from authorised Inlanefreight IT staff

### Pre-Testing Documentation

Two key documents must be signed and verified before testing begins:

1. **Scope of Work (SoW)** - covers testing specifics, methodology, timeline, and agreed deliverables, signed by both company management and an authorised Inlanefreight IT representative
2. **Rules of Engagement (RoE)** - also called an Authorization to Test document, lists all in-scope targets (URLs, IPs, CIDR ranges, credentials if applicable), key personnel from both sides with contact details, and the exact testing start and stop dates along with the allowed testing window

Both documents are signed and complete before testing begins.

### Testing Window and Constraints

- One week for testing, plus two additional days to complete the draft report
- Testing is authorised 24/7
- Heavy vulnerability scans must run outside regular business hours, after 18:00 London time
- Report writing begins during the engagement, not after

### Start of Testing

The tester sets up a testing VM along with a structured note-taking and directory layout before kicking off the engagement. A kickoff email is sent to all relevant personnel to formally signal the start of testing. While initial discovery scans run, the report template is filled in where possible to make efficient use of scan wait time.
