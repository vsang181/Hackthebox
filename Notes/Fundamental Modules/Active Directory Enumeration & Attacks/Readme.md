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

    - [Enumerating Security Controls](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Deeper%20Down%20the%20Rabbit%20Hole/Enumerating%20Security%20Controls.md)
    - [Credentialed Enumeration - from Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Deeper%20Down%20the%20Rabbit%20Hole/Credentialed%20Enumeration%20-%20from%20Linux.md)
    - [Credentialed Enumeration - from Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Initial%20Enumeration/Credentialed%20Enumeration%20-%20from%20Windows.md)
    - [Living Off the Land](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Initial%20Enumeration/Living%20Off%20the%20Land.md)

7. Cooking with Fire

    - [Kerberoasting - from Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Cooking%20with%20Fire/Kerberoasting%20-%20from%20Linux.md)
    - [Kerberoasting - from Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Cooking%20with%20Fire/Kerberoasting%20-%20from%20Windows.md)

8. An ACE in the Hole

    - [Access Control List (ACL) Abuse Primer](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/An%20ACE%20in%20the%20Hole/Access%20Control%20List%20(ACL)%20Abuse%20Primer.md)
    - [ACL Enumeration](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/An%20ACE%20in%20the%20Hole/ACL%20Enumeration.md)
    - [ACL Abuse Tactics](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/An%20ACE%20in%20the%20Hole/ACL%20Abuse%20Tactics.md)
    - [DCSync](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/An%20ACE%20in%20the%20Hole/DCSync.md)
    
9. Stacking The Deck

    - [Privileged Access](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Stacking%20The%20Deck/Privileged%20Access.md)
    - [Kerberos "Double Hop" Problem](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Stacking%20The%20Deck/Kerberos%20%22Double%20Hop%22%20Problem.md)
    - [Bleeding Edge Vulnerabilities](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Stacking%20The%20Deck/Bleeding%20Edge%20Vulnerabilities.md)
    - [Miscellaneous Misconfigurations](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Active%20Directory%20Enumeration%20%26%20Attacks/Stacking%20The%20Deck/Miscellaneous%20Misconfigurations.md)

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
