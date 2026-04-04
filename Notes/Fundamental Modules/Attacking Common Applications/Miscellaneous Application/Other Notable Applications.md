## Other Notable Applications

A solid penetration testing methodology transfers to any application, not just the nine covered in this module. The core pattern is always the same: identify the technology and version, check for default credentials, look for built-in admin functionality that can be weaponised, and search for known public exploits. The applications below are commonly encountered in corporate environments and deserve attention during an assessment.

***

## Quick Reference: Attack Surfaces

| Application | Default Credentials to Try | Primary Attack Vector |
|---|---|---|
| Apache Axis2 | `admin:axis2` | AAR file upload webshell via admin console |
| IBM WebSphere | `system:manager` | WAR file deployment via admin console |
| Oracle WebLogic | `weblogic:weblogic1` | Java deserialization RCE via T3/IIOP protocol |
| Zabbix | `Admin:zabbix` | RCE via Zabbix agent commands or API abuse |
| Nagios | `nagiosadmin:PASSW0RD` | RCE via command injection or outdated CVEs |
| VMware vCenter | `administrator@vsphere.local` | Unauthenticated file upload RCE (CVE-2021-22005) |
| DotNetNuke (DNN) | `host:dnnhost` | Auth bypass, file upload, directory traversal |
| Elasticsearch | None by default | Unauthenticated API data access on exposed port 9200 |

***

## High-Value Targets in Detail

### Apache Axis2

Axis2 is a Java web services framework often found sitting on top of a Tomcat installation. If Tomcat itself is hardened, check whether Axis2 is accessible at `/axis2/` since it may have its own admin panel with separate default credentials (`admin:axis2`). With admin access, upload a webshell packaged as an `.aar` file (Axis2 service archive). The Metasploit module `exploit/multi/http/axis2_deployer` handles this automatically.

### IBM WebSphere

WebSphere has accumulated a long history of critical CVEs. If the admin console is accessible (typically on port 9060 or 9443), try the default credentials `system:manager`. A successful login allows WAR file deployment via the Applications section, identical to the Tomcat exploitation path.

### Oracle WebLogic

WebLogic carries 190+ reported CVEs, with the majority being Java deserialization vulnerabilities.  These affect the T3 and IIOP protocols exposed on the default port 7001. The attack sends a crafted serialized Java object to the listener, which deserialises it without validation and executes embedded OS commands. Notable examples include CVE-2019-2729 (CVSS 9.8) and CVE-2025-0789 (CVSS 9.8), both of which are unauthenticated.  The `ysoserial` tool generates the payloads, and Metasploit modules exist for several versions. Always check if the T3 port is exposed before attempting credential-based attacks.

### Zabbix

Zabbix is an open-source network and infrastructure monitoring platform that has suffered SQL injection, authentication bypass, stored XSS, LDAP password disclosure, and multiple RCE vulnerabilities. Beyond known CVEs, it also has built-in functionality that can be abused. The Zabbix API allows authenticated users to create items that execute arbitrary commands on monitored hosts via the Zabbix agent, which can be used to gain a shell on any monitored system the agent account can reach.

### Nagios

Nagios is another monitoring platform worth fingerprinting carefully. Check for the default credential `nagiosadmin:PASSW0RD` and identify the exact version, as several versions are vulnerable to unauthenticated RCE and root privilege escalation through command injection in plugin handling. Any version from Nagios XI 5.6.x to 5.7.x should be cross-referenced against the CVE list before moving on.

### VMware vCenter

vCenter is present in almost every large enterprise and is one of the highest-impact targets on an internal network. A full compromise of vCenter gives access to the console of every virtual machine it manages. [CVE-2021-22005](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-22005) is a critical unauthenticated arbitrary file upload in the Analytics service (CVSS 9.8). An attacker with network access to port 443 can upload a specially crafted file and achieve RCE with root privileges, regardless of how vCenter is configured.  It affects vCenter 6.5, 6.7, and 7.0 and is listed in CISA's Known Exploited Vulnerabilities catalogue.

Post-exploitation value is exceptional. On the Windows appliance variant, privilege escalation is straightforward using token impersonation tools like JuicyPotato. Instances running as `SYSTEM` or as a domain admin account have been observed in real assessments, making vCenter a potential single path to full domain compromise.

### Elasticsearch

Elasticsearch by default exposes its REST API on port 9200 with no authentication. Older versions are vulnerable to remote code execution via the dynamic scripting engine (see [Exploit-DB 36337](https://www.exploit-db.com/exploits/36337)). Even without RCE, an unauthenticated `GET /_search?q=*` query against a forgotten internet-facing instance returns the entire document corpus, which frequently contains PII, API keys, and internal credentials.

### DotNetNuke (DNN)

DNN is a .NET CMS with a history of auth bypass, directory traversal, stored XSS, arbitrary file download, and file upload bypass vulnerabilities. Fingerprint it via the `/DNN/` or `/DesktopModules/` URL paths and the `X-Powered-By: ASP.NET` header. The admin account password resets are handled client-side in older versions, enabling trivial bypass.

***

## Wikis and Intranets

Internal wikis (MediaWiki, Confluence, SharePoint) and custom intranet pages are frequently overlooked during assessments but carry significant risk. Beyond known platform CVEs, the primary value is content enumeration. Document repositories and internal search functionality regularly surface:

- Credentials embedded in runbooks and IT documentation
- SSH private keys in shared code snippets
- Database connection strings in deployment guides
- VPN or service account passwords in onboarding documents

SharePoint in particular has a powerful search index. A low-privileged authenticated user can often retrieve far more sensitive documents than they were ever intended to access simply by using the built-in search.

***

## Applying the Methodology to Unknown Applications

When an application is not on any known list, the approach is the same:

1. Fingerprint the technology via HTTP headers, error messages, file extensions, and URL patterns
2. Search for the product name in Searchsploit, Exploit-DB, and the NVD
3. Attempt default or common credentials before attempting any exploit
4. Explore built-in admin functionality such as file upload, task scheduling, and script execution
5. Look for configuration files, connection strings, or API keys reachable from the authenticated session

Most compromises during large assessments come not from sophisticated zero-days but from default credentials on a forgotten Tomcat or Nagios instance discovered on page 340 of an EyeWitness report.
