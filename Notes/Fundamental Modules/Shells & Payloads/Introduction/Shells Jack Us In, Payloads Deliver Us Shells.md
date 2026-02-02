## Shells Jack Us In, Payloads Deliver Us Shells

![nonono](https://github.com/user-attachments/assets/fdbc6dd2-602d-491f-83b4-7b94981ea67b)

A **shell** is a program that gives a user an interface to send instructions to an operating system and view output (for example Bash, Zsh, `cmd.exe`, and PowerShell). In penetration testing and offensive security, getting a shell is often the outcome of exploiting a vulnerability or bypassing a control to gain interactive (or semi-interactive) access to a host.

You’ll often hear phrases like:

- “I caught a shell.”
- “I popped a shell!”
- “I dropped into a shell!”
- “I’m in!”

In practice, these usually mean the tester achieved code execution and obtained some level of remote command execution on the target OS. Most of this module focuses on what happens *after* enumeration and selecting a promising exploit: turning execution into a usable session and then leveraging that access safely and effectively.

### Why get a shell?

A shell gives you direct access to the OS, system commands, and the filesystem. Once you have it, you can start post-exploitation work such as local enumeration, privilege escalation discovery, credential hunting, lateral movement/pivoting, and file transfer.

Without a shell (or at least reliable command execution), you’re typically limited to what the vulnerable service exposes, which makes it harder to validate impact or proceed to deeper objectives.

A shell can also provide longer working time and better tooling ergonomics. That said, persistence is a sensitive action: in real engagements it should be explicitly permitted by scope and rules of engagement, because it changes risk and cleanup requirements.

Shell access is often CLI-based rather than graphical (VNC/RDP), which can reduce bandwidth and sometimes reduce the obviousness of activity. This is not a guarantee of stealth, since command execution can still be logged (PowerShell logging, Sysmon, bash history, EDR telemetry) and network callbacks can be noisy.

Being comfortable in the command line also improves speed and repeatability. You can automate tasks with scripts, chain commands, and reduce manual clicking, which helps both productivity and operational consistency.

### How we’ll use “shell” in this module

| Perspective | Description |
|---|---|
| Computing | The text-based userland environment used to administer tasks and run commands (Bash, Zsh, `cmd.exe`, PowerShell). |
| Exploitation & Security | The result of exploiting a vulnerability or bypassing controls to gain remote command execution and interactive access (example: gaining a Windows command prompt after successful exploitation, such as an SMB RCE scenario like EternalBlue in legacy environments). |
| Web | A web shell is a server-side script (often dropped via file upload or code injection) that lets an operator run commands, browse files, and perform actions through HTTP requests, often via a browser or scripted client. |

In hands-on work, you’ll also see common shell/session characteristics discussed alongside the term “shell,” such as:

- Interactive vs. non-interactive sessions (can you run full-screen programs, handle prompts, and get clean output?)
- TTY/PTY quality (for example upgrading a dumb reverse shell into a fully interactive PTY on Linux)
- Reverse vs. bind connectivity (who initiates the connection, and what does the network allow?)
- Stability and privilege context (which user, which process, which integrity level)

### Web shell examples

A few common web shell “families” you’ll encounter (often named after the server-side technology they target) include:

- PHP web shells (deployed on PHP-capable servers such as Apache/Nginx with PHP-FPM; frequently seen on LAMP/LEMP stacks)
- ASP.NET web shells (deployed on IIS; typically implemented as `.aspx` pages running under the application pool identity)
- JSP web shells (deployed on Java application servers such as Apache Tomcat; commonly implemented as `.jsp` files)

Control is usually performed by sending HTTP requests that include a command or task parameter, then reading the response body for output. Depending on how the server is configured, a web shell may execute with limited permissions (for example, a restricted service account) and still require privilege escalation to fully control the host.

### Payloads deliver us shells

In general IT usage, “payload” can mean different things:

| Context | Meaning |
|---|---|
| Networking | The data portion carried by a packet, separate from headers and protocol metadata. |
| Basic computing | The part of an instruction or message that represents the “useful work” once framing/headers are removed. |
| Programming | The data carried/processed by a routine or instruction (the part you actually intend to act on). |
| Exploitation & Security | Code or a sequence of actions delivered to a target to achieve an effect (often code execution, a beacon, or a shell), and the term is also used broadly in malware discussions. |

In this module, we use “payload” in the exploitation/security sense: the code (or command sequence) we deliver to a vulnerable target to achieve remote access and establish a shell session. We’ll cover different payload types and delivery methods, and how to choose appropriately based on target OS, architecture, available interpreters, application constraints, and network/defensive controls.
