# Laudanum, One Webshell to Rule Them All

**[Laudanum](https://github.com/jbarcia/Web-Shells/tree/master/laudanum)** is a curated collection of ready-made injectable web shell files covering multiple server-side languages, designed specifically for use during penetration tests where a file upload vector has been identified. Rather than writing shells from scratch, Laudanum provides tested, functional templates that can be copied, minimally modified, and deployed immediately. It is pre-installed on both Kali Linux and Parrot OS at `/usr/share/laudanum`; on other distributions it can be installed with `sudo apt install laudanum` or cloned directly from the repository.

## Directory Structure

The Laudanum repository is organised by target technology stack:

| Directory | Target Environment |
|---|---|
| `/usr/share/laudanum/asp` | Classic ASP on Windows IIS; includes `cmd.asp`, `shell.asp`, and `upload.asp` |
| `/usr/share/laudanum/aspx` | ASP.NET on modern Windows IIS; includes `shell.aspx` for reverse shell and command execution |
| `/usr/share/laudanum/jsp` | Java Server Pages on Tomcat, JBoss, and similar; includes `cmd.jsp` and `shell.jsp` |
| `/usr/share/laudanum/php` | PHP on Apache or Nginx; includes `cmd.php`, `shell.php`, `upload.php`, `portscan.php`, and database interaction scripts |
| `/usr/share/laudanum/cfm` | ColdFusion environments; includes `cmd.cfm` |
| `/usr/share/laudanum/wordpress` | WordPress-specific injectable files |

Most files can be copied and used as-is for simple command execution. Shell files that establish reverse connections require modification before use to insert the operator's IP address.

## Preparing a Shell for Deployment

Never deploy Laudanum files directly from `/usr/share/laudanum` without first making a working copy. Modifying the original files risks breaking the template for future use. Copy the relevant file to a working directory:

```bash
cp /usr/share/laudanum/aspx/shell.aspx /home/tester/demo.aspx
```

Open the copied file and insert the operator's IP address into the `allowedIps` variable (line 59 in `shell.aspx`). This access control variable restricts which IP addresses can interact with the deployed shell, preventing opportunistic access by third parties if the file is discovered during the engagement window.

Two further preparation steps improve operational security before upload:

- Remove the ASCII art banner and all comments from the file. These elements are commonly used as detection signatures by AV engines and web application firewalls; their presence can trigger an alert before the shell is ever executed.
- Rename the output file to something contextually plausible for the target application, reducing the likelihood of the filename appearing suspicious in server logs or to a reviewing administrator.

## Deployment: ASPX Shell on IIS

This walkthrough targets a Windows IIS host running an ASP.NET web application at `status.inlanefreight.local`. Add the target to `/etc/hosts` before proceeding:

```
<TARGET_IP> status.inlanefreight.local
```

The target application presents a file upload function at the bottom of its status page. Select the prepared `demo.aspx` file and submit the upload. A successful upload returns the server-side path where the file was saved, for example:

```
\\files\demo.aspx
```

Navigate to the shell using that path. This particular application uses a backslash separator in the disclosed path rather than a forward slash; entering `status.inlanefreight.local\\files\demo.aspx` into the browser causes it to normalise the URL automatically to:

```
http://status.inlanefreight.local//files/demo.aspx
```

Not all applications will preserve the original filename on upload or expose a public files directory. Some implementations randomise filenames, restrict directory browsing, or apply server-side extension filtering. When these controls are in place, additional bypass techniques are required before the shell can be reached.

## Issuing Commands Through the Shell

Once the shell is accessible in the browser, commands are submitted through the Laudanum interface and their output is rendered on the page. Standard Windows system enumeration commands are issued directly:

```
systeminfo
whoami
ipconfig /all
net user
```

The shell session operates under the identity of the IIS application pool account, typically `IIS APPPOOL\DefaultAppPool` or `NT AUTHORITY\NETWORK SERVICE` on default installations. As with all web shells, this is the starting context for privilege escalation; enumerate `whoami /priv` and local group membership early to assess available escalation paths.
