# Introduction to Active Directory

You will encounter Active Directory in most corporate environments. Because it is feature-rich and highly interconnected, it also exposes a large and often misunderstood attack surface. If you are studying penetration testing or working in information security, you need a solid grasp of how Active Directory works before you can assess, attack, or defend it effectively.

To work confidently with Active Directory, you must understand its core building blocks, how objects are structured, and how authentication, authorisation, and access control are enforced across a domain. Weak design choices, poor configuration, or incorrect assumptions can quickly turn these mechanisms into security issues.

These notes focus on the fundamentals you need to build that foundation. You will work through key Active Directory concepts, common terminology, important components, and the protocols that make a domain function. The goal is to give you enough clarity and context so that later work on enumeration, abuse, and exploitation of Active Directory environments makes technical sense rather than feeling like guesswork.

## Table of Contents

1. Active Directory Overview
  - Why Active Directory?
  - Active Directory Research Over the Years

2. Active Directory Fundamentals
  - Active Directory Structure
  - Active Directory Terminology
  - Active Directory Objects
  - Active Directory Functionality

3. Active Directory Protocols
  - Kerberos, DNS, LDAP, MSRPC
  - NTLM Authentication

4. All About Users
  - User and Machine Accounts
  - Active Directory Groups
  - Active Directory Rights and Privileges

5. Digging in Deeper
  - Security in Active Directory
  - Examining Group Policy
    
