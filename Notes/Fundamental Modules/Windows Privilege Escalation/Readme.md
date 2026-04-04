# Windows Privilege Escalation

Windows privilege escalation is the process of moving from an initial low-privilege foothold to NT AUTHORITY\SYSTEM or a local/domain administrator account. Unlike Linux where root is a single target, Windows presents multiple privilege tiers: standard user, local admin, SYSTEM, and domain admin, each requiring different techniques and opening different doors.

# Table of Contents

1. Introduction

    - Introduction to Windows Privilege Escalation
    - Useful Tools

2. Getting the lay of teh Land

    - Situational Awareness
    - Initial Enumeration
    - Communication with Process

3. Windows User Privileges

    - Windows Privileges Overview
    - Selmpersonate and SeAssignPrimaryToken
    - SeDebugprivilege
    - SeTakeOwnershipPrivlege

4. Windows Group Privileges

    - Windows Built-in Groups
    - Evnet Log Readers
    - DnsAdmins
    - Hyper-V Administrators
    - Print Operators
    - Server Operators

5. Attacking the OS

    - User Account Control
    - Weak Permissions
    - Kernel Exploits
    - Vulnerable Services
    - DLL Injection

6. Credntial Theft

    - Credntial Hunting
    - Other Files
    - Further Credntial Theft

7. Restricted Environments

    - Critix Breakout

8. Additional techniques

    - Interacting with Users
    - Pillaging
    - Miscellaneous Techniques

9. Dealing with End of Life Systems

    - legacy Operating Systems
    - Windows Server
    - Windows Desktop Versions

10. Closing Thoughts

    - Windows Hardening
