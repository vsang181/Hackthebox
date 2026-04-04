# Linux Privilege Escalation

Privilege escalation on Linux refers to the process of gaining a higher level of access than what was initially obtained, typically moving from a low-privileged shell (such as www-data or a standard user account) toward root. During a penetration test, this phase determines whether initial access translates into full system compromise, credential harvesting, or a pivot point into the wider network.

# Table of Contents

1. Introduction

    - [Introduction to Linux Privilege Escalation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Introduction/Introduction%20to%20Linux%20Privilege%20Escalation.md)

2. Information Gathering

    - [Environment Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Information%20Gathering/Environment%20Enumeration.md)
    - [Linux Services & Internals Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Information%20Gathering/Linux%20Services%20%26%20Internals%20Enumeration.md)
    - [Credential Hunting](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Information%20Gathering/Credential%20Hunting.md)

3. Environment-based Privilege Escalation

    - path Abuse
    - Wildcard Abuse
    - Escaping Restricted Shells

4. Permissions-based Privilege Escalation

    - Special Permissions
    - Sudo Rights Abuse
    - Privileged Groups
    - Capabilities

5. Service-based Privilege Escalation

    - Vulnerable Services
    - Cron Job Abuse
    - LXD
    - Docker
    - Kubernetes
    - Logrotate
    - Miscellaneous techniques

6. Linux Internals-based Privilege Escalation

    - Kernel Exploits
    - Shared Libraries
    - Shared Object Hijacking
    - Python Library Hijacking

7. Recent 0-Days

    - Sudo
    - Polkit
    - Dirty Pipe
    - Netfilter

8. Hardening Considerations

    - Linux Hardening
