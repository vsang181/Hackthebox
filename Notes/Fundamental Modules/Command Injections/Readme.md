# Command Injection

Command injection is a critical vulnerability class where user-supplied input is passed unsanitised to a system shell, allowing arbitrary OS commands to execute on the server with the privileges of the web server process. It consistently ranks among the most severe vulnerabilities in the OWASP Top 10 and can lead to full server compromise in a single step.

## Table of Contents

1. Introduction

    - [Intro of Command Injections](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Introduction/Intro%20to%20Command%20Injections.md)

2. Exploitation

    - [Detection](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Exploitation/Detection.md)
    - [Injecting Commands](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Exploitation/Injecting%20Commands.md)
    - [Other Injection Operators](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Exploitation/Other%20Injection%20Operators.md)

3. Filter Evasion

    - [Identifying Filters](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Identifying%20Filters.md)
    - [Bypassing Space Filters](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Bypassing%20Space%20Filters.md)
    - [Bypassing Other Blacklisted Characters](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Bypassing%20Other%20Blacklisted%20Characters.md)
    - [Bypassing Blacklisted Commands](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Bypassing%20Blacklisted%20Commands.md)
    - [Advanced Command Obfuscation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Advanced%20Command%20Obfuscation.md)
    - [Evasion Tools](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Filter%20Evasion/Evasion%20Tools.md)

4. Prevention

    - Command Injection Prevention
