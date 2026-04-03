# Attacking Common Applications

Penetration testers regularly encounter the same applications across different client environments. An application that is secure in one deployment may be misconfigured or unpatched in the next, making it essential to understand how to enumerate and attack each one consistently.

Common targets include:

- Content Management Systems (CMS)
- Custom web applications
- Internal developer and sysadmin portals
- API services and middleware

Building a firm methodology for identifying and exploiting these applications means that even unfamiliar targets can be approached systematically, since most share underlying technologies, frameworks, or misconfigurations with something you have seen before.

## Table of Contents

1. Setting the Stage

    - Introduction to Attacking Common Applications
    - Application Discovery & Enumeration

2. Content Management Systems (CMS)

    - WordPress - Discovery & Enumeration
    - Attacking WordPress
    - Joomla - Discovery & Enumeration
    - Attacking Joomla
    - Drupal - Discovery & Enumeration
    - Attacking Drupal

3. Servlet Containers/SOftware Development

    - Tomcat- Discovery & Enumeration
    - Attacking Tomcat
    - Jenkins - Discovery & Enumeration
    - Attacking Jenkins

4. Infrastructure/Network Monitoring Tools

    - Splunk - Discovery & Enumeration
    - Attacking Splunk
    - PRTG Network Monitor

5. Custtomer Service Mgmt & Configuration Management

    - osTicket
    - Gitlab - Discovery & Enumeration
    - Attacking Gitlab

6. Common gateway Interface

    - Attacking Tomcat CGI
    - Attacking CGI Applications - Shellshock

7. Thick Client Application

    - Attacking Thick Client Applications
    - Exploiting Web Vulnerabilities in Thick-Client Applications

8. Miscellaneous Application

    - ColdFusion - Discovery & Enumeration
    - Attacking ColdFusion
    - IIS Tilde Enumeration
    - Attacking LDAP
    - Web Mass Assignment Vulnerabilities
    - Attacking Applications Connecting to Services
    - Other Notable Applications

9. Closing Out

    - Application Hardening
