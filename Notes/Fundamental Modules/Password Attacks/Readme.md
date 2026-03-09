# Password Attacks

Passwords remain the most widespread authentication mechanism in corporate environments and, by extension, the most frequently targeted credential type during penetration tests. Despite decades of security guidance, weak and reused passwords continue to drive a substantial proportion of breaches: approximately 81% of hacking-related corporate breaches involve weak or reused credentials, and data from Verizon's 2025 DBIR shows that only 3% of passwords meet NIST complexity requirements.

## Table of Contents

1. Introduction

   - [Introduction](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Introduction/Introduction.md)

2. Password Cracking Techniques

   - [Introduction to Password Cracking](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Introduction%20to%20Password%20Cracking.md)
   - [Introduction to John The Ripper](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Introduction%20to%20John%20The%20Ripper.md)
   - [Introduction to Hashcat](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Introduction%20to%20Hashcat.md)
   - [Writing Custom Wordlists and Rules](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Writing%20Custom%20Wordlists%20and%20Rules.md)
   - [Cracking Protected Files](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Cracking%20Protected%20Files.md)
   - [Cracking Protected Archives](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20Cracking%20Techniques/Cracking%20Protected%20Archives.md)

3. Remote Password Attacks

   - [Network Services](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Remote%20Password%20Attacks/Network%20Services.md)
   - [Spraying, Stuffing, and Defaults](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Remote%20Password%20Attacks/Spraying%2C%20Stuffing%2C%20and%20Defaults.md)

4. Extracting Passwords from Windows Systems

   - [Windows Authentication Process](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Windows%20Authentication%20Process.md)
   - [Attacking SAM, SYSTEM, and SECURITY](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Attacking%20SAM%2C%20SYSTEM%2C%20and%20SECURITY.md)
   - [Attacking LSASS](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Attacking%20LSASS.md)
   - [Attacking Windows Credential Manager](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Attacking%20Windows%20Credential%20Manager.md)
   - [Attacking Active Directory and NTDS.dit](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Attacking%20Active%20Directory%20and%20NTDS.md)
   - [Credential Hunting in Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Windows%20Systems/Credential%20Hunting%20in%20Windows.md)

5. Extracting Passwords from Linux Systems

   - [Linux Authentication Process](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Linux%20Systems/Linux%20Authentication%20Process.md)
   - [Credential Hunting in Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20Linux%20Systems/Credential%20Hunting%20in%20Linux.md)

6. Extracting Passwords from the Network

   - [Credential Hunting in Network Traffic](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20the%20Network/Credential%20Hunting%20in%20Network%20Traffic.md)
   - [Credential Hunting in Network Shares](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Extracting%20Passwords%20from%20the%20Network/Credential%20Hunting%20in%20Network%20Shares.md)

7. Windows Lateral Movement Techniques

   - [Pass the Hash (PtH)](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Windows%20Lateral%20Movement%20Techniques/Pass%20the%20Hash%20(PtH).md)
   - [Pass the Ticket (PtT) from Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Windows%20Lateral%20Movement%20Techniques/Pass%20the%20Ticket%20(PtT)%20from%20Windows.md)
   - [Pass the Ticket (PtT) from Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Windows%20Lateral%20Movement%20Techniques/Pass%20the%20Ticket%20(PtT)%20from%20Linux.md)
   - [Pass the Certificate](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Windows%20Lateral%20Movement%20Techniques/Pass%20the%20Certificate.md)

8. Password Management
   
   - [Password Policies](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20management/Password%20Policies.md)
   - [Password Managers](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Password%20Attacks/Password%20management/Password%20Managers.md)
