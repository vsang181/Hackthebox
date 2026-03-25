# Web Fuzzing

Web fuzzing is the process of automatically sending a large number of requests to a web server, substituting values from a wordlist into a specific part of the request, and using the server's responses to determine what exists. It is the primary method for discovering pages, directories, parameters, and other assets that are not publicly linked or documented.

***

## How It Works

The logic is simple. A web server returns different HTTP status codes depending on whether a requested resource exists:

- **200 OK**: The page or resource exists and is accessible
- **301/302**: The resource exists but redirects to another location
- **403 Forbidden**: The resource exists but access is denied
- **404 Not Found**: The resource does not exist

By sending hundreds of requests per second and recording the response code for each, ffuf can map out a significant portion of a web application's content in minutes. Resources that are never linked to from any page are just as discoverable this way as those that are, which is what makes fuzzing essential for thorough enumeration.

***

## Wordlists

The quality of your fuzzing results depends heavily on the wordlist you use. Rather than building wordlists from scratch, the [SecLists](https://github.com/danielmiessler/SecLists) repository on GitHub is the standard resource for web fuzzing wordlists. It is pre-installed on PwnBox at:

```
/opt/useful/SecLists/
```

For directory and page fuzzing, the go-to wordlist is `directory-list-2.3`, available in small, medium, and large variants:

```bash
locate directory-list-2.3-small.txt
# /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt
```

The small variant is a good starting point for initial reconnaissance as it runs quickly. If you get few results, escalate to medium and then large for wider coverage.

SecLists is organised into categories covering every fuzzing scenario you will encounter:

| Category | Path |
|---------|------|
| Web content discovery | `Discovery/Web-Content/` |
| DNS and subdomains | `Discovery/DNS/` |
| Usernames | `Usernames/` |
| Passwords | `Passwords/` |
| Fuzzing payloads | `Fuzzing/` |

***

## Running Your First Ffuf Scan

The basic directory fuzzing command places `FUZZ` at the position in the URL where directory names should be tested:

```bash
ffuf -u http://SERVER_IP:PORT/FUZZ \
     -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt \
     -ic
```

Flag breakdown:

- `-u`: The target URL with `FUZZ` marking the injection point
- `-w`: Path to the wordlist
- `-ic`: Ignore comments in the wordlist (removes copyright header lines that would otherwise be sent as payloads and clutter results)

***

## Reading the Output

Ffuf outputs a line for each result that did not match its default filters, showing the payload, status code, response size, word count, and response time:

```
admin                   [Status: 200, Size: 1234, Words: 45, Lines: 30]
login                   [Status: 302, Size: 0, Words: 0, Lines: 0]
uploads                 [Status: 403, Size: 276, Words: 20, Lines: 10]
```

All three results above are worth investigating even though only one returns 200. A 302 redirect means the resource exists and is actively routing traffic somewhere. A 403 means the directory exists but has access restrictions in place, which may be bypassable through other techniques.

***

## Handling Noise

On some applications, every request returns a 200 response regardless of whether the resource exists (custom 404 pages). In these cases, filter by response size instead of status code. Run one request to a path you know does not exist, note the response size, and exclude it:

```bash
ffuf -u http://SERVER_IP:PORT/FUZZ \
     -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt \
     -ic \
     -fs 4242
```

Where `4242` is the byte size of the false-positive response. This is one of the most important ffuf skills to develop, since unfiltered output on a noisy application is nearly unreadable.
