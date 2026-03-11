# Service Misconfigurations

Service misconfigurations are one of the most consistently exploitable categories of vulnerability encountered during assessments. Unlike software flaws that require a specific version or patch level, misconfigurations are the result of human decisions -- and they appear in every environment regardless of how modern the infrastructure is. [Security Misconfiguration ranks fifth on the OWASP Top 10](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/) precisely because it is so pervasive.

## Authentication Weaknesses

### Default Credentials

Historically, many services shipped with hardcoded default credentials and administrators would leave them unchanged after deployment. While modern software increasingly forces credential creation during installation, default credentials remain common in older applications, embedded devices, and network appliances. A short internet search for a specific product often reveals its default username and password immediately.

Once a service banner is identified, checking for known default credentials should be the first manual step. If none are documented, common weak combinations are worth attempting before moving to any automated approach:

```
admin:admin
admin:password
admin:<blank>
root:12345678
administrator:Password
```

Password policies discussed in the previous sections apply here directly -- administrators must be required to set credentials that meet a minimum complexity standard when configuring any new service or device, not just user accounts.

### Anonymous Authentication

Some services are deliberately or accidentally configured to allow authentication without credentials. Anonymous access that should be restricted to a read-only public directory sometimes extends to the entire service due to a single misconfigured directive. FTP anonymous login, SMB null sessions, and LDAP anonymous binds are the most common examples encountered during internal assessments. From an attacker's perspective, anonymous access is immediately actionable -- there is no credential barrier to begin enumerating content.

## Misconfigured Access Rights

Credential access is only one part of the problem. An account may authenticate correctly but be granted far more access than its role requires. An FTP account intended only for uploading files to a single directory might have read access to the entire server, exposing configuration files, plaintext credentials, PII, or internal documentation belonging to other departments.

This category of misconfiguration scales with the size of the organisation -- the more accounts exist, the harder it is to audit each one's permissions manually. Two widely adopted frameworks for structuring access rights correctly are:

- [Role-based access control (RBAC)](https://en.wikipedia.org/wiki/Role-based_access_control) -- permissions are assigned to roles, and users are assigned to roles, rather than permissions being granted directly to individual accounts
- [Access control lists (ACL)](https://en.wikipedia.org/wiki/Access-control_list) -- explicit lists define which principals can perform which operations on which resources

A detailed comparison of both approaches and their tradeoffs is available in [Choosing the best access control strategy](https://authress.io/knowledge-base/role-based-access-control-rbac) by Warren Parad from Authress. The core principle that applies regardless of which model is used is least privilege -- every account should have only the permissions required to perform its specific function and nothing more.

## Unnecessary Defaults

Software and devices ship with default configurations optimised for usability and ease of deployment, not security. Features that are enabled by default to ensure the product works out of the box often expand the attack surface unnecessarily in a production environment. The [OWASP Top 10 identifies](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/) the following as the most common default-related issues:

- Unnecessary features, services, ports, pages, accounts, or privileges that are enabled but not required
- Default accounts and passwords that remain active and unchanged after installation
- Error handling that exposes stack traces or overly detailed diagnostic messages to end users
- Security features from previous versions that remain disabled after a system upgrade rather than being reviewed and enabled

Each of these represents a configuration decision that was never actively made -- the administrator accepted the default rather than deliberately locking down the deployment.

## Preventing Misconfiguration

The [OWASP guidance](https://owasp.org/Top10/A05_2021-Security_Misconfiguration/) for preventing security misconfiguration centres on treating configuration as a repeatable, audited process rather than a one-time setup task. Practically, this means:

- Building a repeatable hardening process that can deploy a locked-down environment consistently, with development, QA, and production environments configured identically using different credentials per environment
- Starting from a minimal platform -- unused features, components, documentation, and sample files should be removed or never installed
- Treating configuration review as part of the patch management cycle, not a separate activity
- Segmenting application architecture so that a misconfiguration in one component does not expose others, using network segmentation, containerisation, or cloud security groups
- Running automated scans and audits regularly to detect drift from the approved baseline before an attacker does

Specific controls that should be applied to any service before it is placed in production include: disabling admin interfaces that do not need to be externally accessible, turning off debugging output, removing default credentials, restricting directory listing on file servers, and ensuring security-related HTTP headers are sent to clients where applicable.

The pattern encountered throughout this module holds here too -- a weak password on an SMB share, an anonymously accessible FTP directory, or an MSSQL instance running under a domain admin account are all misconfigurations rather than software bugs. They are correctable, detectable through routine auditing, and entirely preventable with deliberate configuration management.  
