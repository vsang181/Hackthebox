# Password Attacks

Passwords remain the most widespread authentication mechanism in corporate environments and, by extension, the most frequently targeted credential type during penetration tests. Despite decades of security guidance, weak and reused passwords continue to drive a substantial proportion of breaches: approximately 81% of hacking-related corporate breaches involve weak or reused credentials, and data from Verizon's 2025 DBIR shows that only 3% of passwords meet NIST complexity requirements.

## Table of Contents

1. Introduction

- Introduction

2. Password Cracking Techniques

- Introduction to Password Cracking
- Introduction to John The Ripper
- Introduction to Hashcat
- Writing Custom Wordlists and Rules
- Cracking Protected Files
- Cracking Protected Archives

3. Remote Password Attacks

- Network Services
- Spraying, Stuffing, and Defaults

4. Extracting Passwords from Windows Systems

- Windows Authentication Process
- Attacking SAM, SYSTEM, and SECURITY
- Attacking LSASS
- Attacking Windows Credential Manager
- Attacking Active Directory and NTDS.dit
- Credential Hunting in Windows

5. Extracting Passwords from Linux Systems

- Linux Authentication Process
- Credential Hunting in Linux

6. Extracting Passwords from the Network

- Credential Hunting in Network Traffic
- Credential Hunting in Network Shares

7. Windows Lateral Movement Techniques

- Pass the Hash (PtH)
- Pass the Ticket (PtT) from Windows
- Pass the Ticket (PtT) from Linux
- Pass the Certificate

8. Password Management
   
- Password Policies
- Password Managers

