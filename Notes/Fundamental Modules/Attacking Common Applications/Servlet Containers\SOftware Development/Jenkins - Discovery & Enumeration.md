## Jenkins Discovery and Enumeration

[Jenkins](https://www.jenkins.io/) is an open-source continuous integration and automation server written in Java, originally released in 2005 under the name Hudson before being renamed in 2011 following a dispute with Oracle.  Over 86,000 companies use Jenkins, including Facebook, Netflix, LinkedIn, and Robinhood, primarily to automate building, testing, and deploying software. It is a critical piece of infrastructure in most development pipelines and a high-value target during both internal and external penetration tests. 

***

## Why Jenkins Matters During Pentests

Jenkins is particularly attractive as a target for several reasons:

- It is very commonly installed on Windows servers running as the `SYSTEM` account, meaning code execution through Jenkins often yields the highest possible privilege level on the machine
- On Linux, it frequently runs as `root` or a highly privileged service account
- A compromised Jenkins instance in an Active Directory environment provides an immediate foothold for domain enumeration
- Internal deployments are regularly misconfigured with no authentication enabled or left on default credentials

***

## Default Ports and Services

| Port | Purpose |
|---|---|
| `8080` | Default web UI and HTTP API, runs on Tomcat/Jetty |
| `5000` / `50000` | JNLP agent port used for master-slave communication |

 Jenkins can be placed behind a reverse proxy on port 80 or 443, so these default ports may not always be directly visible during scanning. 

***

## Footprinting and Identification

Jenkins has a distinctive login page that makes it easy to fingerprint visually. The login path is `/login` and the page typically displays the Jenkins logo and branding. A `Server` header or page source reference to Jenkins will confirm the platform. Browsing to `/configureSecurity/` when authentication is not enforced will directly expose the security configuration panel, which is a clear sign of a misconfigured instance.

Other indicators:

- Page title contains `Jenkins`
- Paths such as `/about`, `/oops`, `/people`, and `/asynchPeople` are Jenkins-specific endpoints
- The `/robots.txt` on Jenkins instances typically disallows crawling of `/` entirely

***

## Authentication Modes

Jenkins supports several authentication backends: 

- Jenkins internal user database (default)
- LDAP
- Unix user database
- Delegated to the servlet container
- No authentication at all

The default installation uses the Jenkins internal database and does not allow public account registration. However, misconfigurations are common, and two particularly dangerous settings to check are:

- "Allow users to register" which lets anyone create an account
- "Enable anonymous read permission" which grants unauthenticated users read access to the entire instance

***

## Enumeration Checklist

When you land on a Jenkins instance, work through these checks in order:

1. Try default credentials: `admin:admin`, `admin:password`, `jenkins:jenkins`
2. Check if the login page is present at all, as some instances have no authentication
3. Browse to `/configureSecurity/` to inspect the current security configuration
4. Check `/people` or `/asynchPeople` to enumerate registered user accounts
5. Look for the Jenkins version number in the page footer or `/about` page
6. Note the version and cross-reference against known CVEs

A particularly critical recent vulnerability worth noting is [CVE-2024-23897](https://nvd.nist.gov/vuln/detail/CVE-2024-23897), an unauthenticated arbitrary file read affecting Jenkins versions up to 2.441. It exploits the CLI's `args4j` parser, which misinterprets `@` prefixed arguments as file read directives. Unauthenticated attackers can read the first few lines of any file on the controller filesystem, while users with `Overall/Read` permission can read files in full. This has been actively exploited in ransomware campaigns. 
