# Shells & Payloads

Choosing the right shell and payload isn’t just about “getting a session”—it can be the difference between being detected during a penetration test and maintaining **access** long enough to complete your objectives. This module covers common ways to establish an interactive shell on a target host (for example, reverse vs. bind shells, PTY upgrades, and web shells) and how to generate payloads that match the target’s constraints.

You’ll learn how to align payloads with the environment: operating system and CPU architecture (for example, $$x86$$ vs. $$x64$$, ARM), process context, available interpreters (PowerShell, Bash, Python), and network realities such as inbound filtering and outbound egress rules. We’ll also touch on practical trade-offs like staged vs. stageless payloads, payload size, stability, and detection surface (AV/EDR, logging, and noisy network indicators), so you can select an approach that is both effective and appropriate for the engagement.

## Table of Contents

1. Introduction
  - Shells Jack Us In, Payloads Deliver Us Shells
  - CAT5 Security's Engagement Preparation
  
2. Shell Basics
  - Anatomy of a Shell
  - Bind Shells
  - Reverse Shells

3. Payloads
  - Introduction to Payloads
  - Automating Payloads & Delivery with Metasploit
  - Crafting Payloads with MSFvenom

4. Windows Shells
  - Infiltrating Windows
  
5. NIX Shells
  - Infiltrating Unix/Linux
  - Spawning Interactive Shells  

6. Web Shells
  - Introduction to Web Shells
  - Laudanum, One Webshell to Rule Them All
  - Antak Webshell
  - PHP Web Shells

7. Additional Considerations
  - Detection & Prevention
