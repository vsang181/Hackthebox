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

    - [Introduction to Attacking Common Applications](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Setting%20the%20Stage/Introduction%20to%20Attacking%20Common%20Applications.md)
    - [Application Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Setting%20the%20Stage/Application%20Discovery%20%26%20Enumeration.md)

2. Content Management Systems (CMS)

    - [WordPress - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/WordPress%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking WordPress](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/Attacking%20WordPress.md)
    - [Joomla - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/Joomla%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Joomla](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/Attacking%20Joomla.md)
    - [Drupal - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/Drupal%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Drupal](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Content%20Management%20Systems%20(CMS)/Attacking%20Drupal.md)

3. Servlet Containers/SOftware Development

    - [Tomcat- Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Servlet%20Containers%5CSOftware%20Development/Tomcat%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Tomcat](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Servlet%20Containers%5CSOftware%20Development/Attacking%20Tomcat.md)
    - [Jenkins - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Servlet%20Containers%5CSOftware%20Development/Jenkins%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Jenkins](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Servlet%20Containers%5CSOftware%20Development/Attacking%20Jenkins.md)
      
4. Infrastructure/Network Monitoring Tools

    - [Splunk - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Infrastructure%5CNetwork%20Monitoring%20Tools/Splunk%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Splunk](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Infrastructure%5CNetwork%20Monitoring%20Tools/Attacking%20Splunk.md)
    - [PRTG Network Monitor](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Infrastructure%5CNetwork%20Monitoring%20Tools/PRTG%20Network%20Monitor.md)

5. Custtomer Service Mgmt & Configuration Management

    - [osTicket](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Custtomer%20Service%20Mgmt%20%26%20Configuration%20Management/Attacking%20GitLab.md)
    - [Gitlab - Discovery & Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Custtomer%20Service%20Mgmt%20%26%20Configuration%20Management/Gitlab%20-%20Discovery%20%26%20Enumeration.md)
    - [Attacking Gitlab(https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Attacking%20Common%20Applications/Custtomer%20Service%20Mgmt%20%26%20Configuration%20Management/Attacking%20GitLab.md)

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
