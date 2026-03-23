# Active Directory Enumeration & Attacks

Active Directory is the backbone of identity and access management in the vast majority of enterprise Windows environments. It acts as the single source of truth for who users are, what they are allowed to access, and how that access is enforced across every machine, service, and application joined to the domain. Because virtually everything in a Windows enterprise runs through AD in some form, compromising it does not just mean owning one system -- it typically means owning the entire organization.

# Table of Contents

1. Setting The Stage

    - [Introduction to Active Directory Enumeration & Attacks](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Setting%20The%20Stage/Introduction%20to%20Active%20Directory%20Enumeration%20%26%20Attacks.md)
    - [Tools of the Trade](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Setting%20The%20Stage/Tools%20of%20the%20Trade.md)

2. Initial Enumeration

    - [External Recon and Enumeration Principles](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Initial%20Enumeration/External%20Recon%20and%20Enumeration%20Principles.md)
    - [Initial Enumeration of the Domain](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Initial%20Enumeration/Initial%20Enumeration%20of%20the%20Domain.md)

3. Sniffing out a Foothold

    - [LLMNR/NBT-NS Poisoning - from Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Sniffing%20out%20a%20Foothold/LLMNR%5CNBT-NS%20Poisoning%20-%20from%20Linux.md)
    - [LLMNR/NBT-NS Poisoning - from Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Sniffing%20out%20a%20Foothold/LLMNR%5CNBT-NS%20Poisoning%20-%20from%20Windows.md)
      
4. Sighting In, Hunting for a User

    - [Password Spraying Overview](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Sighting%20In%2C%20Hunting%20for%20a%20User/Password%20Spraying%20Overview.md)
    - [Enumerating & Retrieving Password Policies](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Sighting%20In%2C%20Hunting%20for%20a%20User/Enumerating%20%26%20Retrieving%20Password%20Policies.md)
    - [Password Spraying - Making a Target User List](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Sighting%20In%2C%20Hunting%20for%20a%20User/Password%20Spraying%20-%20Making%20a%20Target%20User%20List.md)

5. Spray Responsibly

    - [Internal Password Spraying - from Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Spray%20Responsibly/Internal%20Password%20Spraying%20-%20from%20Linux.md)
    - [Internal Password Spraying - from Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Spray%20Responsibly/Internal%20Password%20Spraying%20-%20from%20Windows.md)

6. Deeper Down the Rabbit Hole

    - Enumerating Security Controls
    - Credentialed Enumeration - from Linux
    - Credentialed Enumeration - from Windows
    - Living Off the Land

7. Cooking with Fire

    - Kerberoasting - from Linux
    - Kerberoasting - from Windows

8. An ACE in the Hole

    - Access Control List (ACL) Abuse Primer
    - ACL Enumeration
    - ACL Abuse Tactics
    - DCSync
    
9. Stacking The Deck

    - Privileged Access
    - Kerberos "Double Hop" Problem
    - Bleeding Edge Vulnerabilities
    - Miscellaneous Misconfigurations

10. Why So trusting

    - Domain Trusts Primer
    - Attacking Domain Trusts - Child -> Parent Trusts - from Windows
    - Attacking Domain Trusts - Child -> Parent Trusts - from Linux

11. Breaking Down Boundaries

    - Attacking Domain Trusts - Cross-Forest Trust Abuse - from Windows
    - Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux

12. Defensive Considerations

    - Hardening Active Directory
    - Additional AD Auditing Techniques
