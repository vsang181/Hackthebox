# Shells & Payloads

Choosing the right shell and payload isn’t just about “getting a session”—it can be the difference between being detected during a penetration test and maintaining **access** long enough to complete your objectives. This module covers common ways to establish an interactive shell on a target host (for example, reverse vs. bind shells, PTY upgrades, and web shells) and how to generate payloads that match the target’s constraints.

You’ll learn how to align payloads with the environment: operating system and CPU architecture (for example, $$x86$$ vs. $$x64$$, ARM), process context, available interpreters (PowerShell, Bash, Python), and network realities such as inbound filtering and outbound egress rules. We’ll also touch on practical trade-offs like staged vs. stageless payloads, payload size, stability, and detection surface (AV/EDR, logging, and noisy network indicators), so you can select an approach that is both effective and appropriate for the engagement.

## Table of Contents

1. [Introduction](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Introduction)
  - [Shells Jack Us In, Payloads Deliver Us Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Introduction/Shells%20Jack%20Us%20In%2C%20Payloads%20Deliver%20Us%20Shells.md)
  - [CAT5 Security's Engagement Preparation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Introduction/CAT5%20Security's%20Engagement%20Preparation.md)
  
2. [Shell Basics](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Shell%20Basics)
  - [Anatomy of a Shell](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Shell%20Basics/Anatomy%20of%20a%20Shell.md)
  - [Bind Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Shell%20Basics/Bind%20Shells.md)
  - [Reverse Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Shell%20Basics/Reverse%20Shells.md)

3. [Payloads](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Payloads)
  - [Introduction to Payloads](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Payloads/Introduction%20to%20Payloads.md)
  - [Automating Payloads & Delivery with Metasploit](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Payloads/Automating%20Payloads%20%26%20Delivery%20with%20Metasploit.md)
  - [Crafting Payloads with MSFvenom](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Payloads/Crafting%20Payloads%20with%20MSFvenom.md)

4. [Windows Shells](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Windows%20Shells)
  - [Infiltrating Windows](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Windows%20Shells/Infiltrating%20Windows.md)
  
5. [NIX Shells](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/NIX%20Shells)
  - [Infiltrating Unix/Linux](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/NIX%20Shells/Infiltrating%20Unix%5CLinux.md)
  - [Spawning Interactive Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/NIX%20Shells/Spawning%20Interactive%20Shells.md)  

6. [Web Shells](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Web%20Shells)
  - [Introduction to Web Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Web%20Shells/Introduction%20to%20Web%20Shells.md)
  - [Laudanum, One Webshell to Rule Them All](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Web%20Shells/Laudanum%2C%20One%20Webshell%20to%20Rule%20Them%20All.md)
  - [Antak Webshell](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Web%20Shells/Antak%20Webshell.md)
  - [PHP Web Shells](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Web%20Shells/PHP%20Web%20Shells.md)

7. [Additional Considerations](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Additional%20Considerations)
  - [Detection & Prevention](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Shells%20%26%20Payloads/Additional%20Considerations/Detection%20%26%20Prevention.md)
