## Introduction to Attacking Common Applications

Web applications appear in almost every environment a penetration tester will encounter, and the same applications show up repeatedly across different organisations. A deployment that is fully patched in one environment may be running an outdated version or using default credentials in the next. The goal is to understand how these applications work deeply enough to identify both known vulnerabilities and ways to abuse built-in functionality, not just reproduce known exploits.

***

## What Are Web Applications

Web applications run on a client-server architecture where front-end components execute in the user's browser and back-end components run on the server, typically backed by a database. All types, whether commercial, open-source, or custom-built, are susceptible to the vulnerability classes listed in the [OWASP Top 10](https://owasp.org/www-project-top-ten/), including SQL injection, XSS, remote code execution, local file read, and unrestricted file upload. Understanding how to abuse built-in functionality is just as important as knowing public CVEs, since hardened applications often still expose dangerous features to authenticated users.

***

## Why Web Applications Matter as Targets

As organisations harden their network perimeters and reduce exposed services, web applications have become a primary attack surface for both malicious actors and penetration testers. A compromised application on the external perimeter can serve as an initial foothold, while an internal application can enable lateral movement or expose sensitive data. Research commissioned by [Barracuda](https://www.barracuda.com/) found that 72% of surveyed organisations experienced at least one breach caused by an application vulnerability, with 32% suffering two breaches and 14% suffering three.

***

## Application Categories

The table below lists the application categories and specific tools commonly encountered during assessments:

| Category | Applications |
|---|---|
| Web Content Management | [WordPress](https://wordpress.org/), [Joomla](https://www.joomla.org/), [Drupal](https://www.drupal.org/), DotNetNuke |
| Application Servers | [Apache Tomcat](https://tomcat.apache.org/), Oracle WebLogic, IBM WebSphere, Phusion Passenger |
| SIEM | [Splunk](https://www.splunk.com/), LogRhythm, Trustwave |
| Network Management | [PRTG Network Monitor](https://www.paessler.com/prtg), ManageEngine OpManager |
| IT Management | [Nagios](https://www.nagios.org/), [Zabbix](https://www.zabbix.com/), Puppet, ManageEngine ServiceDesk Plus |
| Software Frameworks | [JBoss](https://www.redhat.com/en/technologies/jboss-middleware), Axis2 |
| Customer Service | [osTicket](https://osticket.com/), [Zendesk](https://www.zendesk.com/) |
| Search Engines | [Elasticsearch](https://www.elastic.co/), [Apache Solr](https://solr.apache.org/) |
| SCM and Code Repos | [Atlassian JIRA](https://www.atlassian.com/software/jira), [GitLab](https://about.gitlab.com/), [GitHub](https://github.com/), [Bitbucket](https://bitbucket.org/), Bugzilla |
| Dev Tools | [Jenkins](https://www.jenkins.io/), [Atlassian Confluence](https://www.atlassian.com/software/confluence), [phpMyAdmin](https://www.phpmyadmin.net/) |
| Enterprise Integration | Oracle Fusion Middleware, BizTalk Server, [Apache ActiveMQ](https://activemq.apache.org/) |

[WordPress](https://enlyft.com/tech/products/wordpress) alone accounts for nearly 70% of the web content management market share across over 3.7 million tracked companies. [Splunk](https://enlyft.com/tech/products/splunk) holds close to 30% of the SIEM market. Even applications with a smaller share are worth studying because the attack mindset transfers across unfamiliar targets.

***

## Core Applications Covered

| Application | Description |
|---|---|
| [WordPress](https://wordpress.org/) | Open-source PHP CMS running on Apache with MySQL. Highly customisable through themes and plugins, which frequently introduce third-party vulnerabilities |
| [Drupal](https://www.drupal.org/) | Open-source PHP CMS supporting MySQL, PostgreSQL, or SQLite. Extensible through modules and themes |
| [Joomla](https://www.joomla.org/) | Open-source PHP CMS typically using MySQL. Considered the third most used CMS globally after WordPress and Shopify |
| [Apache Tomcat](https://tomcat.apache.org/) | Open-source Java web server hosting Java Servlets and JSP. Widely used with frameworks like Spring |
| [Jenkins](https://www.jenkins.io/) | Open-source Java automation server running in servlet containers. Has a history of unauthenticated RCE vulnerabilities |
| [Splunk](https://www.splunk.com/) | Log analytics and SIEM platform. Often holds sensitive data; notable CVEs include [CVE-2018-11409](https://nvd.nist.gov/vuln/detail/CVE-2018-11409) and [CVE-2011-4642](https://nvd.nist.gov/vuln/detail/CVE-2011-4642) |
| [PRTG Network Monitor](https://www.paessler.com/prtg) | Agentless network monitoring using ICMP, WMI, SNMP, and NetFlow. Written in Delphi |
| [osTicket](https://osticket.com/) | Open-source support ticketing system written in PHP, running on Apache or IIS with MySQL |
| [GitLab](https://about.gitlab.com/) | Source code platform with CI/CD, issue tracking, and code review. Offers community and enterprise editions |

***

## Real-World Application: Mindset in Practice

One example of applying this mindset is the [Nexus Repository OSS](https://www.sonatype.com/products/repository-oss) application from Sonatype. On first encounter with an unpatched version, default credentials of `admin:admin123` were still active. Authenticated API access led to remote code execution. On a second engagement with the same application, the API was locked down, but the [Tasks](https://help.sonatype.com/en/tasks.html) feature was enabled, allowing a [Groovy](https://groovy-lang.org/) script written in Java syntax to achieve the same result. A similar pattern applies to [ManageEngine OpManager](https://www.manageengine.com/products/applications_manager/me-opmanager-monitoring.html), which allows script execution under the application's service account, often `NT AUTHORITY\SYSTEM`.

The pattern is consistent: find what the application can do natively, then determine whether an authenticated or unauthenticated user can weaponise that functionality.

***

## Lab Environment Setup

Module exercises use virtual hosts (Vhosts) to simulate a multi-server environment. All Vhosts map to the same physical host, so you must add manual entries to `/etc/hosts` before accessing labs via FQDN.

Add all required hostnames in a single command:

```bash
IP=10.129.42.195
printf "%s\t%s\n\n" "$IP" "app.inlanefreight.local dev.inlanefreight.local blog.inlanefreight.local" | sudo tee -a /etc/hosts
```

Verify the entries were added correctly:

```bash
cat /etc/hosts
```

Expected output at the bottom of the file:

```
10.129.42.195   app.inlanefreight.local dev.inlanefreight.local blog.inlanefreight.local
```

- Sections that use only an IP address (such as Splunk exercises) do not require a hosts file entry
- If a FQDN is unreachable after spawning a target, always check and update `/etc/hosts` first
- Each section that requires Vhosts will list the hostnames to add at the bottom of the page
