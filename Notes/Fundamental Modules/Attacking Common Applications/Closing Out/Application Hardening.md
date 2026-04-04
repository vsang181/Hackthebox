## Application Hardening

The starting point for any hardening programme is an accurate, up-to-date application inventory covering both internal and external-facing services. Without knowing what exists, nothing can be protected. Tools like Nmap and EyeWitness can build this inventory on a budget. The audit often surfaces shadow IT, deprecated applications no longer needed, and misconfigurations like unlicensed Splunk instances that have silently dropped authentication requirements.

***

## Universal Hardening Principles

These controls apply to every web application regardless of function or vendor:

- Change all default credentials immediately after installation and disable default admin accounts where the platform allows it
- Enforce strong password policies and require MFA for all administrative accounts at minimum; extend it to all users where possible
- Apply the principle of least privilege across roles and service accounts so that compromise of any single account limits blast radius
- Restrict admin login pages to internal networks or specific IP ranges, removing them from public internet access unless there is a documented business requirement
- Apply vendor patches promptly; high-severity fixes should be applied within 14 days
- Configure automated backups to a secondary location so recovery from compromise does not depend on the compromised system
- Deploy a WAF as a supplementary layer after the above controls are in place, not as a substitute for them
- Integrate applications with Active Directory via LDAP/SSO to centralise identity management, reduce credential sprawl, and make auditing practical
- Perform regular penetration tests and follow through on remediation. Recurring vulnerabilities of the same class indicate a process gap, not just a technical one

***

## Application-Specific Hardening

### WordPress

Use the [WordFence](https://www.wordfence.com/) plugin to add a firewall, malware scanning, login attempt throttling, country blocking, and built-in 2FA.  Disable the PHP file editor in `wp-config.php` with `define('DISALLOW_FILE_EDIT', true)` to prevent code execution if an admin account is compromised. Force HTTPS for the admin panel with `define('FORCE_SSL_ADMIN', true)`. Protect the XML-RPC endpoint, which is the largest bruteforce target in WordPress, by requiring 2FA for it or disabling it entirely if unused.
### Joomla

Install the [AdminExile](https://extensions.joomla.org/extension/adminexile/) plugin to require a secret key appended to the admin URL:

```
http://joomla.example.com/administrator?yoursecretkey
```

Any request to `/administrator` without the key returns a 404, removing the login page from the public attack surface.

### Drupal

Hide or relocate the user login page. The default `/user/login` path is well-known and is the first place an attacker looks. Drupal provides [official guidance](https://www.drupal.org/docs/7/managing-users/hide-user-login) for moving it to a custom path.

### Apache Tomcat

Restrict the Manager and Host-Manager applications to `localhost` only in `conf/context.xml`. If external access is genuinely required, enforce IP whitelisting using the `RemoteAddrValve` and set a non-default username with a strong password. Never leave these interfaces exposed to the internet. The Manager application in particular gives any authenticated user the ability to deploy a WAR file and achieve RCE.

### Jenkins

Configure granular access control using the [Matrix Authorization Strategy plugin](https://plugins.jenkins.io/matrix-auth). Without it, Jenkins either allows all authenticated users full control or requires no authentication at all. Assign specific permissions per user and per project, and block the Jenkins script console from all accounts that do not require it.

### Splunk

Ensure a valid licence is in place. Splunk in trial or free mode removes authentication entirely, leaving the full search and admin interface open to anyone who can reach port 8000. Change the default `admin:changeme` credential immediately after installation and enforce Splunk's built-in role-based access controls.

### PRTG Network Monitor

Stay current with patches and change the default credential `prtgadmin:prtgadmin` at installation. Earlier versions stored credentials and admin passwords in cleartext backup files left in the `ProgramData` directory.

### osTicket

Restrict the admin panel and staff login to internal networks. There is no business reason for a support ticketing system's admin interface to be publicly accessible. Ensure submitted ticket content cannot carry HTML or script injection that would execute in staff member browsers.

### GitLab

Enforce admin approval for new account registrations and configure allowed email domains to prevent outsiders from registering. Audit all public repositories and groups. A self-hosted GitLab instance with open self-registration and public repositories effectively hands attackers source code, SSH keys, API tokens, and internal network information with no effort required.

***

## Exposure Minimisation Checklist

Before deploying any application, answer these questions:

- Does this application need to be reachable from the internet, or only internally?
- Does every feature the application exposes serve an active business purpose?
- Are all default accounts, credentials, and sample content removed?
- Is the admin interface on the same port and path as the public application?
- Are logs being collected and reviewed, or only stored?
- Is there a tested recovery process, or just a backup that has never been restored?

Most successful compromises during penetration tests do not involve sophisticated exploits. They involve a forgotten Tomcat Manager with `tomcat:tomcat`, a GitLab instance with self-registration enabled, or a PRTG install that has never had its password changed. Hardening is fundamentally about eliminating the easy wins before an attacker finds them.
