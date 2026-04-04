## GitLab Discovery and Enumeration

[GitLab](https://about.gitlab.com/) is a web-based Git repository platform providing source control, wikis, issue tracking, and CI/CD pipelines. It is available as a free community edition and a paid enterprise version, and is widely deployed as a self-hosted instance within company infrastructure. During penetration tests, GitLab is highly valuable even without a direct vulnerability, since repositories frequently contain hardcoded credentials, API keys, SSH private keys, configuration files, and internal infrastructure details.

***

## Identification

GitLab is easy to spot. Browsing to the instance URL presents the unmistakable GitLab login page with the GitLab logo. Beyond visual identification:

- The URL path `/users/sign_in` confirms the GitLab login page
- The path `/explore` shows public projects without authentication
- The `/users/sign_up` registration page may be accessible even when registration is disabled
- The `/help` page, accessible when logged in, reveals the exact GitLab version number

The only reliable way to fingerprint the version without authentication is through a low-risk version disclosure exploit. Without a version number, avoid running exploits blindly. Focus on secret hunting instead.

***

## User Enumeration

GitLab leaks valid usernames through two separate mechanisms that persist even in recent versions:

### Registration Form

Attempting to register with an existing username returns an error indicating the username is taken. Attempting to register with an existing email returns:

```
1 error prohibited this user from being saved: Email has already been taken
```

This works even when the Sign-up enabled checkbox is unchecked in the admin settings. The `/users/sign_up` page remains accessible regardless of that setting. 

### GraphQL API

[CVE-2021-4191](https://nvd.nist.gov/vuln/detail/CVE-2021-4191) is an unauthenticated user enumeration vulnerability in the GitLab GraphQL API affecting versions 13.0 through 14.8.2.  A missing authentication check on certain GraphQL queries allows any unauthenticated attacker to collect registered GitLab usernames, full names, and email addresses from private instances with restricted sign-ups.  This was patched in versions 14.8.2, 14.7.4, and 14.6.5 in February 2022. 

***

## Known RCE Exploits by Version

If you can determine the version, cross-reference these known exploits before hunting manually:

| Version | Exploit |
|---|---|
| GitLab 11.4.7 | RCE via Exiftool | [Exploit-DB 49257](https://www.exploit-db.com/exploits/49257) |
| GitLab 12.9.0 | RCE | [Exploit-DB 48431](https://www.exploit-db.com/exploits/48431) |
| GitLab CE 13.9.3 | RCE | [Exploit-DB 49944](https://www.exploit-db.com/exploits/49944) |
| GitLab CE 13.10.2 | RCE | [Exploit-DB 49951](https://www.exploit-db.com/exploits/49951) |
| GitLab CE 13.10.3 | User enumeration + RCE | [Exploit-DB 49821](https://www.exploit-db.com/exploits/49821) |

***

## Repository Enumeration

Work through these steps in order based on your access level:

### Without Authentication

Browse to `/explore` to view any public projects. Public repositories may contain:

- Infrastructure configuration files
- Scripts with hardcoded credentials or API keys
- Application source code reviewable for vulnerabilities
- SSH private keys accidentally committed

Commit history is equally important: deleted files remain accessible in the git history, and a secret removed from a recent commit can still be read from an earlier one. 

### After Registering an Account

If the GitLab instance allows open registration, creating an account typically reveals internal repositories marked as visible to authenticated users. Look for:

- Internal wikis describing production systems
- CI/CD pipeline definitions (`.gitlab-ci.yml`) containing environment variables with secrets
- Deployment keys and access tokens in pipeline configuration
- Development or staging environment configuration files

A recent scan of 5.6 million public GitLab repositories using TruffleHog uncovered 17,430 verified live secrets across 2,800 organisations, including GCP credentials, MongoDB keys, and over 400 GitLab tokens within GitLab-hosted projects.  The pattern of secrets appearing on the same platform they authenticate is common and worth looking for specifically. 

### Using Obtained Credentials

If credentials are found through OSINT sources like [Dehashed](https://dehashed.com/) or credential dumps, test them against the GitLab staff login. Two-factor authentication is disabled by default on GitLab instances, so credential reuse often works directly with no additional bypass needed.

***

## Mitigation Checklist

Controls that prevent the attacks described above:

- Enforce 2FA across all user accounts
- Use Fail2Ban or similar to block brute-force login attempts
- Restrict GitLab access to internal IP ranges unless external access is required
- Enable GitLab's native Secret Detection scanner to scan commit history for leaked secrets
- Enable secret push protection to block commits containing secrets before they land
- Treat the registration page as an attack surface and restrict it to company email domains
