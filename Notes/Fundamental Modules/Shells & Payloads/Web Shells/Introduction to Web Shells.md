# Introduction to Web Shells

Web servers represent one of the most consistently accessible and exploitable elements of an organisation's attack surface, particularly during external penetration tests where perimeter hardening typically eliminates direct exposure of services such as SMB or RDP. The shift of software services to browser-accessible, HTTP/S-based platforms means web applications now constitute the primary entry point in the majority of external assessments. Gaining a foothold through a web application, whether via file upload vulnerabilities, SQL injection, command injection, or misconfigured services, is the standard path into an otherwise well-defended network perimeter.

## What Is a Web Shell?

A web shell is a browser-based shell session that provides interactive access to the underlying operating system of a web server through a malicious script uploaded to the server. The script is written in the server-side language of the target web application, executes within the web server process, and accepts commands through HTTP requests, returning output directly in the browser. Gaining remote code execution through a web shell requires first identifying a vulnerability or misconfiguration that permits file upload to a web-accessible directory.

Common vectors for web shell upload include:

- Unrestricted file upload forms that accept arbitrary file types without validation
- Authenticated application functionality where file uploads are permitted, such as user profile picture upload areas, where client-side checks can be bypassed
- Application administration panels, including **[Apache Tomcat](https://tomcat.apache.org/)** Manager, **[Oracle WebLogic](https://www.oracle.com/middleware/technologies/weblogic.html)**, and **[Axis2](https://axis.apache.org/axis2/java/core/)**, which support WAR file deployment containing JSP code as part of their intended functionality
- Misconfigured FTP services that allow authenticated or anonymous uploads directly into the server's web root
- Remote file inclusion (RFI) and local file inclusion (LFI) vulnerabilities that can be chained to achieve code execution

## Operational Characteristics

Web shells operate entirely within the web server's process context, which defines both their capabilities and their limitations. Commands execute under the permissions of the web server account (typically `www-data`, `apache`, or `NETWORK SERVICE`), meaning privilege escalation is required to move beyond the web server's access boundaries. Some applications are configured to purge uploaded files after a defined period, making the web shell session inherently transient. For this reason, a web shell is best treated as the initial foothold mechanism that enables upgrading to a more stable reverse or bind shell rather than a long-term interactive access channel.

Web shells are also observable: each command is issued as an HTTP request, meaning web server access logs will record every interaction. Operators should be aware of this logging behaviour and, where appropriate, consider whether log entries are being reviewed in real time by the target organisation's security operations function.

## Shell Persistence Consideration

The instability of web shells as a standalone access method makes the upgrade path important. Once remote code execution is confirmed through the web shell, the next step is to use that execution capability to deliver and trigger a reverse shell payload, transitioning from browser-based interaction to a proper interactive shell session on the operator's listener. The web shell establishes the conditions for that delivery; the reverse shell provides the reliable, persistent access channel needed for deeper enumeration and post-exploitation activity.

The following sections cover specific web shell implementations across PHP, ASPX, and JSP targets, with practical examples of deployment and command execution for each technology stack.
