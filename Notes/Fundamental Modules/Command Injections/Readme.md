# Command Injection

Command injection is a critical vulnerability class where user-supplied input is passed unsanitised to a system shell, allowing arbitrary OS commands to execute on the server with the privileges of the web server process. It consistently ranks among the most severe vulnerabilities in the OWASP Top 10 and can lead to full server compromise in a single step.

## Table of Contents

1. Introduction

    - Intro of Command Injections

2. Exploitation

    - Detection
    - Injecting Commands
    - Other Injection Operators

3. Filter Evasion

    - Identifying Filters
    - Bypassing Space Filters
    - Bypassing Other Blacklisted Characters
    - Bypassing Blacklisted Commands
    - Advanced Command Obfuscation
    - Evasion Tools

4. Prevention

    - Command Injection Prevention
