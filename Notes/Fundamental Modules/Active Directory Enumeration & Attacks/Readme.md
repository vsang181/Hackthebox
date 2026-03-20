# Active Directory Enumeration & Attacks

Active Directory is the backbone of identity and access management in the vast majority of enterprise Windows environments. It acts as the single source of truth for who users are, what they are allowed to access, and how that access is enforced across every machine, service, and application joined to the domain. Because virtually everything in a Windows enterprise runs through AD in some form, compromising it does not just mean owning one system -- it typically means owning the entire organization.

# Table of Contents

1. Setting The Stage

    - Introduction to Active Directory Enumeration & Attacks
    - Tools of the Trade
    - Tools of the Trade

2. Initial Enumeration

    - External Recon and Enumeration Principles
    - Initial Enumeration of the Domain

3. Sniffing out a Foothold

    - Initial Enumeration of the Domain
    - LLMNR/NBT-NS Poisoning - from Windows
      
4. Sighting In, Hunting for a User

    - Password Spraying Overview
    - Enumerating & Retrieving Password Policies
    - Password Spraying - Making a Target User List

5. Spray Responsibly

    - Internal Password Spraying - from Linux
    - Internal Password Spraying - from Windows

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
