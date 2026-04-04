# Windows Privilege Escalation

Windows privilege escalation is the process of moving from an initial low-privilege foothold to NT AUTHORITY\SYSTEM or a local/domain administrator account. Unlike Linux where root is a single target, Windows presents multiple privilege tiers: standard user, local admin, SYSTEM, and domain admin, each requiring different techniques and opening different doors.

# Table of Contents

1. Introduction

    - [Introduction to Windows Privilege Escalation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Introduction/Introduction%20to%20Windows%20Privilege%20Escalation.md)
    - [Useful Tools](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Introduction/Useful%20Tools.md)

2. Getting the lay of teh Land

    - [Situational Awareness](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Getting%20the%20lay%20of%20teh%20Land/Situational%20Awareness.md)
    - [Initial Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Getting%20the%20lay%20of%20teh%20Land/Initial%20Enumeration.md)
    - [Communication with Process](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Getting%20the%20lay%20of%20teh%20Land/Communication%20with%20Processes.md)

3. Windows User Privileges

    - [Windows Privileges Overview](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20User%20Privileges/Windows%20Privileges%20Overview.md)
    - [Selmpersonate and SeAssignPrimaryToken](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20User%20Privileges/SeImpersonate%20and%20SeAssignPrimaryToken.md)
    - [SeDebugprivilege](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20User%20Privileges/SeDebugPrivilege.md)
    - [SeTakeOwnershipPrivlege](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20User%20Privileges/SeTakeOwnershipPrivilege.md)

4. Windows Group Privileges

    - [Windows Built-in Groups](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/Windows%20Built-in%20Groups.md)
    - [Evnet Log Readers](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/Event%20Log%20Readers.md)
    - [DnsAdmins](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/DnsAdmins.md)
    - [Hyper-V Administrators](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/Hyper-V%20Administrators.md)
    - [Print Operators](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/Print%20Operators.md)
    - [Server Operators](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Windows%20Group%20Privileges/Server%20Operators.md)

5. Attacking the OS

    - [User Account Control](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Attacking%20the%20OS/User%20Account%20Control.md)
    - [Weak Permissions](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Attacking%20the%20OS/Weak%20Permissions.md)
    - [Kernel Exploits](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Attacking%20the%20OS/Kernel%20Exploits.md)
    - [Vulnerable Services](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Attacking%20the%20OS/Vulnerable%20Services.md)
    - [DLL Injection](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Attacking%20the%20OS/DLL%20Injection.md)

6. Credntial Theft

    - [Credntial Hunting](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Credntial%20Theft/Credential%20Hunting.md)
    - [Other Files](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Credntial%20Theft/Further%20Credential%20Theft.md)
    - [Further Credntial Theft](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Credntial%20Theft/Other%20Files.md)

7. Restricted Environments

    - [Critix Breakout](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Restricted%20Environments/Critix%20Breakout.md)

8. Additional techniques

    - [Interacting with Users](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Additional%20techniques/Interacting%20with%20Users.md)
    - [Pillaging](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Additional%20techniques/Pillaging.md)
    - [Miscellaneous Techniques](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Additional%20techniques/Miscellaneous%20Techniques.md)

9. Dealing with End of Life Systems

    - [Legacy Operating Systems](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Dealing%20with%20End%20of%20Life%20Systems/Legacy%20Operating%20Systems.md)
    - [Windows Server](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Dealing%20with%20End%20of%20Life%20Systems/Windows%20Server.md)
    - [Windows Desktop Versions](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Dealing%20with%20End%20of%20Life%20Systems/Windows%20Desktop%20Versions.md)

10. Closing Thoughts

    - [Windows Hardening](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Windows%20Privilege%20Escalation/Closing%20Thoughts/Windows%20Hardening.md)
