# Parameter Fuzzing - GET

Parameter fuzzing shifts the focus from finding pages and directories to finding the inputs those pages accept. A page that exists but requires a valid parameter to return meaningful content is a common pattern in web applications, and ffuf handles this the same way it handles everything else: place `FUZZ` where the unknown value is.

***

## How GET Parameters Work

GET parameters are appended directly to the URL after a `?` symbol, with multiple parameters separated by `&`:

```
http://admin.academy.htb:PORT/admin/admin.php?param1=value1&param2=value2
```

The structure you are fuzzing is the parameter name, not the value. The value (`key` in the command) is a placeholder that just needs to be something, because the goal at this stage is simply finding which parameter names the application recognises and responds differently to.

***

## The Command

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key \
     -fs xxx
```

Three things to note here:

- `FUZZ` replaces the parameter name, not the value
- The value `key` is arbitrary, its only purpose is to make the request syntactically valid
- `-fs xxx` filters the baseline response size, which you determine by visiting the page once manually before running the scan and noting the response size of the default/unauthorised response

The wordlist `burp-parameter-names.txt` contains parameter names harvested from real-world web applications by Burp Suite, making it well suited for this task compared to a generic directory wordlist.

***

## Interpreting the Result

A hit means the application handled that parameter differently from every other word in the list. That difference shows up as a response size that does not match the baseline. In this case the result is labelled deprecated, which is a common and important finding.

Deprecated parameters are particularly valuable from a security standpoint for a few reasons:

- They were likely removed from the frontend UI and documentation but the backend code still processes them
- They tend to receive less attention during security reviews and patching cycles
- Developers sometimes leave debug or admin parameters in place after removing them from the public interface, assuming nobody will find them
- They may bypass controls that were added to the newer, documented parameters

***

## Workflow Before Running the Scan

Because you need to know the baseline size to set `-fs` correctly, the process is:

1. Visit `admin.academy.htb:PORT/admin/admin.php` manually and note the response size (check browser dev tools or use `curl -s -o /dev/null -w "%{size_download}"`)
2. Or run ffuf briefly without `-fs` and observe that most results share the same size value
3. Then rerun with `-fs [baseline_size]` to filter the noise

```bash
# Quick way to get the baseline size
curl -s -o /dev/null -w "%{size_download}\n" \
     http://admin.academy.htb:PORT/admin/admin.php?test=key
```

***

## GET vs POST Parameter Fuzzing

This section covers GET parameters, but many sensitive parameters are only accepted via POST requests. The distinction matters because:

| Aspect | GET | POST |
|--------|-----|------|
| Parameter location | URL query string | Request body |
| Logged in server logs | Yes, usually | No, usually |
| Used for | Reading data | Submitting data |
| Fuzzing flag in ffuf | Parameter in URL | `-d` flag with body |

The next natural step after finding a GET parameter is checking whether the same endpoint also accepts POST parameters with different or additional functionality, which is what the following section on POST fuzzing covers.
