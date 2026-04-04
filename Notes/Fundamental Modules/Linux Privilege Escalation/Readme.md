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

    - [Path Abuse](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Environment-based%20Privilege%20Escalation/Path%20Abuse.md)
    - [Wildcard Abuse](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Environment-based%20Privilege%20Escalation/Wildcard%20Abuse.md)
    - [Escaping Restricted Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Environment-based%20Privilege%20Escalation/Escaping%20Restricted%20Shells.md)

4. Permissions-based Privilege Escalation

    - [Special Permissions](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Permissions-based%20Privilege%20Escalation/Special%20Permissions.md)
    - [Sudo Rights Abuse](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Permissions-based%20Privilege%20Escalation/Sudo%20Rights%20Abuse.md)
    - [Privileged Groups](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Permissions-based%20Privilege%20Escalation/Privileged%20Groups.md)
    - [Capabilities](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Linux%20Privilege%20Escalation/Permissions-based%20Privilege%20Escalation/Capabilities.md)

5. Service-based Privilege Escalation

    - Vulnerable Services
    - Cron Job Abuse
    - Containers
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
