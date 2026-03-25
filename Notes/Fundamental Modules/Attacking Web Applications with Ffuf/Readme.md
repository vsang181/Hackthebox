# Attacking Web Applications with Ffuf

Ffuf (Fuzz Faster U Fool) is a fast web fuzzer written in Go, designed for web content discovery, parameter fuzzing, subdomain enumeration, and brute-forcing various parts of HTTP requests. Its speed advantage over older tools like dirb and dirbuster, combined with flexible filtering options, has made it a staple in modern web penetration testing and bug bounty workflows.

## Table of Contents

1. Introduction

    - Introduction
    - Web Fuzzing
      
2. Basic Fuzzing

    - Directory Fuzzing
    - Page Fuzzing
    - Recursive Fuzzing

3. Domian Fuzzing

    - DNS Records
    - Sub-domain Fuzzing
    - Vhost Fuzzing
    - Filtering Results

4. Parameter Fuzzing

    - Parameter Fuzzing - GET
    - Parameter Fuzzing - POST
    - Value Fuzzing
