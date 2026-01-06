# HyperText Transfer Protocol (HTTP)

Most applications you interact with every day—both **web** and **mobile**—communicate over the internet using **HTTP**. If you want to understand how web applications work (and how they break), you must first understand this protocol.

HTTP is an **application-layer protocol** used to request and deliver resources on the World Wide Web. The term *hypertext* refers to text that contains links to other resources and is designed to be easily interpreted by humans.

At a high level, HTTP is simple:

* A **client** requests a resource
* A **server** processes the request
* The **server** returns a response

By default, HTTP uses **port 80**, though servers can be configured to listen on other ports.

---

## Client–Server Communication

Every HTTP interaction follows the same basic pattern:

1. You (the client) request a resource
2. The server receives and processes the request
3. The server responds with:

   * A **status code**
   * Optional **headers**
   * Optional **content** (HTML, JSON, files, etc.)

This same process occurs whether you are:

* Loading a webpage
* Submitting a form
* Calling an API
* Downloading a file

---

## URLs and Resource Access

Resources are accessed using a **URL (Uniform Resource Locator)**. A URL contains much more information than just a website name.

<img width="1940" height="480" alt="url_structure" src="https://github.com/user-attachments/assets/82ca62f5-e406-4e35-8dd1-d16c24f8bb60" />

### URL Components

| Component    | Example              | Description                                          |
| ------------ | -------------------- | ---------------------------------------------------- |
| Scheme       | `http://` `https://` | Defines the protocol used                            |
| User Info    | `admin:password@`    | Optional credentials for authentication              |
| Host         | `inlanefreight.com`  | Hostname or IP address                               |
| Port         | `:80`                | Network port (80 for HTTP, 443 for HTTPS by default) |
| Path         | `/dashboard.php`     | Resource being requested                             |
| Query String | `?login=true`        | Parameters passed to the resource                    |
| Fragment     | `#status`            | Client-side reference to a section of the page       |

Only **scheme** and **host** are mandatory. Everything else is optional.

---

## HTTP Flow (High Level)

<img width="2280" height="1041" alt="HTTP_Flow" src="https://github.com/user-attachments/assets/5a64133d-73e9-4421-8a50-48894b6471b5" />

When you enter a URL in your browser:

1. The browser resolves the **domain name** using DNS
2. The DNS server returns an **IP address**
3. The browser sends an HTTP request to the server
4. The server processes the request
5. The server responds (for example, `200 OK`)
6. The browser renders the response

### Important Notes

* Browsers first check the local `/etc/hosts` file before querying DNS
* You can manually map domains to IPs using `/etc/hosts`
* A request to `/` usually returns a default file such as `index.html`

---

## Why HTTP Matters to You

As a penetration tester, HTTP is where:

* User input enters the application
* Authentication happens
* Session tokens are exchanged
* Vulnerabilities are exposed

If you can understand and manipulate HTTP requests, you can:

* Bypass client-side restrictions
* Discover hidden functionality
* Test access controls
* Identify injection points

---

## Using cURL

In this module, you work with two primary tools:

* A **web browser**
* The **cURL** command-line tool

### What Is cURL?

`cURL` (client URL) is a command-line tool used to send requests over HTTP (and many other protocols). It is essential because it allows you to:

* Send raw requests
* Modify headers and payloads
* Automate testing
* Interact with APIs directly

Unlike browsers, cURL does **not render content**. It shows you exactly what the server returns.

---

### Sending a Basic HTTP Request

```bash
curl inlanefreight.com
```

This returns the raw HTML:

```html
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html>
...
```

This is ideal for security testing, where **context matters more than presentation**.

---

### Downloading Files with cURL

Save a remote file using the original filename:

```bash
curl -O inlanefreight.com/index.html
```

Specify your own output filename:

```bash
curl -o page.html inlanefreight.com/index.html
```

Suppress status output:

```bash
curl -s -O inlanefreight.com/index.html
```

---

### Getting Help

View basic help:

```bash
curl -h
```

Useful flags you will use often:

| Flag | Purpose                  |
| ---- | ------------------------ |
| `-d` | Send POST data           |
| `-i` | Include response headers |
| `-o` | Write output to file     |
| `-O` | Use remote filename      |
| `-s` | Silent mode              |
| `-u` | Authentication           |
| `-A` | Custom User-Agent        |
| `-v` | Verbose output           |

For full documentation:

```bash
man curl
```

Or:

```bash
curl --help all
```
