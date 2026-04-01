# Command Injections

Command injection is one of the most critical and directly damaging vulnerability classes in web security. Unlike XSS, which targets users, or LFI, which reads files, command injection executes operating system commands directly on the server, giving an attacker the same level of access as the web server process itself.

## Tabel of Contents

1. Introduction

    - [Intro to File Upload Attacks](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Introduction/Intro%20to%20File%20Upload%20Attacks.md)

2. Basic Exploitation

    - [Absent validation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Basic%20Exploitation/Absent%20Validation.md)
    - [Upload Exploitation](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Command%20Injections/Basic%20Exploitation/Upload%20Exploitation.md)

3. Bypassing Filters

    - Client-Side Validation
    - Black Filters
    - Whitelist Filters
    - Type Filters

4. Other Upload Attacks

    - Limited File Uploads
    - Other Upload Attacks

5. Prevention

    - Preventing File Upload Vulnerabilities
