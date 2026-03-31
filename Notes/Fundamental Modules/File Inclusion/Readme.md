# File Inclusion

File Inclusion is a vulnerability class that arises when a web application dynamically includes files based on user-supplied input without proper validation. Unlike XSS which targets the client, file inclusion targets the server itself, and successful exploitation can escalate all the way to remote code execution.

## Table of Contents

1. Introduction

    - Intro to File Inclusions

2. File Disclosure

    - Local File Inclusion (LFI)
    - Basic Bypasses
    - PHP Filters

3. Remote Code Execution

    - PHP Wrappers
    - Remote File Inclusion (RFI)
    - LFI and File Uploads
    - Log Poisoning

4. Automation and Prevention

    - Automated Scanning
    - File Inclusion Prevention
