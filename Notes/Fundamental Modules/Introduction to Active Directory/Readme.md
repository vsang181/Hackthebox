# Introduction to Active Directory

You will encounter Active Directory in most corporate environments. Because it is feature-rich and highly interconnected, it also exposes a large and often misunderstood attack surface. If you are studying penetration testing or working in information security, you need a solid grasp of how Active Directory works before you can assess, attack, or defend it effectively.

To work confidently with Active Directory, you must understand its core building blocks, how objects are structured, and how authentication, authorisation, and access control are enforced across a domain. Weak design choices, poor configuration, or incorrect assumptions can quickly turn these mechanisms into security issues.

These notes focus on the fundamentals you need to build that foundation. You will work through key Active Directory concepts, common terminology, important components, and the protocols that make a domain function. The goal is to give you enough clarity and context so that later work on enumeration, abuse, and exploitation of Active Directory environments makes technical sense rather than feeling like guesswork.

## Table of Contents

1. [Active Directory Overview](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Overview)
  - [Why Active Directory?](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Overview/Why%20Active%20Directory%3F.md)
  - [Active Directory Research Over the Years](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Overview/Active%20Directory%20Research%20Over%20the%20Years.md)

2. [Active Directory Fundamentals](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Fundamentals)
  - [Active Directory Structure](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Fundamentals/Active%20Directory%20Structure.md)
  - [Active Directory Terminology](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Fundamentals/Active%20Directory%20Terminology.md)
  - [Active Directory Objects](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Fundamentals/Active%20Directory%20Objects.md)
  - [Active Directory Functionality](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Fundamentals/Active%20Directory%20Functionality.md)

3. [Active Directory Protocols](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Protocols)
  - [Kerberos, DNS, LDAP, MSRPC](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Protocols/Kerberos%2C%20DNS%2C%20LDAP%2C%20MSRPC.md)
  - [NTLM Authentication](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Active%20Directory%20Protocols/NTLM%20Authentication.md)

4. [All About Users](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/All%20About%20Users)
  - [User and Machine Accounts](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/All%20About%20Users/User%20and%20Machine%20Accounts.md)
  - [Active Directory Groups](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/All%20About%20Users/Active%20Directory%20Groups.md)
  - [Active Directory Rights and Privileges](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/All%20About%20Users/Active%20Directory%20Rights%20and%20Privileges.md)

5. [Digging in Deeper](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Digging%20in%20Deeper)
  - [Security in Active Directory](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Digging%20in%20Deeper/Security%20in%20Active%20Directory.md)
  - [Examining Group Policy](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Introduction%20to%20Active%20Directory/Digging%20in%20Deeper/Examining%20Group%20Policy.md)
    
