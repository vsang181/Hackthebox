## Splunk Discovery and Enumeration

[Splunk](https://www.splunk.com/) is a log analytics and data visualization platform widely used as a SIEM across large enterprise environments. It stores sensitive security event data, making it a high-value target during penetration tests. Splunk rarely suffers from critical unauthenticated exploits, but built-in functionality combined with weak authentication or the free version's complete lack of authentication makes it a reliable path to code execution on any host running it.

***

## Why Splunk is a Priority Target

- Splunk runs as `root` on Linux and `SYSTEM` on Windows in the majority of deployments
- Every full Splunk installation ships with Python, guaranteeing a scripting capability regardless of the OS
- Admin access provides direct access to scripted inputs, which can run arbitrary system commands
- The platform holds sensitive log data that could reveal credentials, internal network topology, and security controls
- 92 of the Fortune 100 companies use Splunk, making it extremely common in enterprise environments 

***

## Default Ports

| Port | Purpose |
|---|---|
| `8000` | Web UI, served over HTTPS |
| `8089` | Splunk management port and REST API |
| `9997` | Default data receiver for forwarders |
| `8080` | Sometimes used for indexer communication |

An Nmap scan will show both port 8000 and 8089 as `Splunkd httpd`, confirming the instance:

```bash
sudo nmap -sV 10.129.201.50
```

```
8000/tcp open  ssl/http  Splunkd httpd
8089/tcp open  ssl/http  Splunkd httpd
```

***

## The Free Version Problem

The Splunk Enterprise trial converts automatically to the free version after 60 days.  The free version has no authentication whatsoever: there is no login page, no users, no roles, and anyone connecting to port 8000 is passed directly into the web interface as an administrator.  This is a significant security gap that frequently goes unnoticed when a trial is installed and then forgotten. Some organisations also intentionally deploy the free version due to cost, without understanding the implications. 

Indicators that an instance is running the free version:

- No login prompt when browsing to port 8000
- The REST API endpoint `/services/server/info` returns `"isFree":true` in its JSON response
- `"hasLoggedIn":false` in the API response also indicates the default admin password was never changed on Enterprise instances 

## Authentication and Credentials

The default credentials on older Splunk installations are `admin:changeme`, displayed directly on the login page. Newer installations force credential setup during installation. If defaults fail, common passwords worth trying include:

- `admin`
- `Welcome1`
- `Password123`
- `splunk`
- `changeme`

Authentication backends Splunk supports include its own internal database, LDAP, PAM, and scripted authentication. The internal database is the default for most installs.

***

## Paths to Code Execution

Once authenticated, or when facing an unauthenticated free version instance, Splunk offers several built-in mechanisms that lead to code execution: 

- **Scripted inputs**: The primary attack path. Designed to pull data from custom sources, they run any script on STDOUT collection. Since Python is always available, a Python reverse shell can be bundled directly into a Splunk app and deployed as a scripted input.
- **Custom apps and add-ons**: Splunk allows uploading packaged apps via the web UI. A malicious app containing a scripted input with a reverse shell is the standard exploitation technique.
- **Deployment server**: If the Splunk instance is acting as a deployment server with Universal Forwarders checking in, a malicious app pushed from the server will execute on every forwarder host, potentially giving lateral movement across the entire organisation.
- **REST API**: The management port at 8089 exposes the full Splunk REST API, which can also be used to create scripted inputs programmatically

***

## Notable CVEs

Splunk has accumulated over [47 CVEs](https://www.cvedetails.com/vulnerability-list/vendor_id-10963/Splunk.html), but the majority are low-severity or require specific conditions. The most relevant ones for penetration testing:

| CVE | Type | Notes |
|---|---|---|
| CVE-2018-11409 | Information disclosure | Exposes server information without auth |
| CVE-2011-4642 | Authenticated RCE | Very old versions only |
| CVE-2023-46214 | Authenticated RCE | XSLT injection via uploaded file  [uptycs](https://www.uptycs.com/blog/threat-research-report-team/splunk-vulnerability-cve-2023-46214) |
| CVE-2024-36984 | Authenticated RCE | Insecure deserialization on Windows  [sentinelone](https://www.sentinelone.com/vulnerability-database/cve-2024-36984/) |
| CVE-2024-36991 | Path traversal | File read on Windows instances  [orionx.foregenix](https://orionx.foregenix.com/blog/pentesting-exploiting-cve-2024-36991-in-splunk-enterprise) |

The practical focus for Splunk during a pentest is almost always built-in functionality abuse rather than CVE exploitation, since admin access alone gives reliable code execution through scripted inputs regardless of version.
