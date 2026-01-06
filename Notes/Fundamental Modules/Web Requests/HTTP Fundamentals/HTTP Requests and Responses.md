# HTTP Requests and Responses

HTTP communication is fundamentally built around **requests** and **responses**.

An **HTTP request** is initiated by the client (for example, a browser or `curl`) and processed by the server (such as a web server). The request contains all the information the server needs in order to process it, including the requested resource, parameters, headers, and optional data.

Once the server receives and processes the request, it sends back an **HTTP response**. The response includes a status code indicating the result of the request and may include data such as HTML, JSON, images, scripts, or other resources.

---

## HTTP Request

Below is an example of a simple HTTP request:

<img width="1981" height="961" alt="raw_request" src="https://github.com/user-attachments/assets/43671379-9ee7-41d0-b2c7-930a9be777a5" />

```
GET /users/login.html HTTP/1.1
Host: inlanefreight.com
User-Agent: Mozilla/5.0
Cookie: PHPSESSID=c4ggt4jull9obt7aupa55o8vbf
```

This request targets the following URL:

```
http://inlanefreight.com/users/login.html
```

### Request Line Structure

The first line of every HTTP request contains **three fields**, separated by spaces:

| Field   | Example           | Description                                              |
| ------- | ----------------- | -------------------------------------------------------- |
| Method  | GET               | The HTTP verb specifying the action to perform           |
| Path    | /users/login.html | The resource path, optionally including query parameters |
| Version | HTTP/1.1          | The HTTP protocol version                                |

### Headers and Body

* The lines that follow are **HTTP headers**, expressed as key-value pairs (for example `Host`, `User-Agent`, `Cookie`)
* Headers provide metadata and instructions about the request
* Headers are terminated by a blank line
* Optionally, a request body may follow (commonly used with POST or PUT requests)

**Note:**

* HTTP/1.x transmits requests as clear text
* HTTP/2 transmits requests in a binary, dictionary-based format

---

## HTTP Response

After processing the request, the server replies with an HTTP response.

<img width="2020" height="1127" alt="raw_response" src="https://github.com/user-attachments/assets/1bb4fc7f-4cad-47ab-865e-bfd89799b023" />

Example response:

```
HTTP/1.1 200 OK
Date: Tue, 21 Jul 2020 05:20:15 GMT
Server: Apache/2.4.41
Set-Cookie: PHPSESSID=m4u64rqlpfthrvvb12ai9voqgf
Content-Type: text/html; charset=UTF-8
```

### Response Structure

* The **first line** contains:

  * HTTP version
  * Response status code and message (for example `200 OK`)
* This is followed by **response headers**
* A blank line separates headers from the **response body**
* The body may contain:

  * HTML
  * JSON
  * Images
  * Scripts
  * Documents such as PDFs

---

## Viewing Requests and Responses with cURL

By default, `curl` only prints the response body. To view **both the full request and response**, use the `-v` (verbose) flag:

```
curl inlanefreight.com -v
```

Example output excerpt:

```
> GET / HTTP/1.1
> Host: inlanefreight.com
> User-Agent: curl/7.65.3
> Accept: */*

< HTTP/1.1 401 Unauthorized
< Date: Tue, 21 Jul 2020 05:20:15 GMT
< Server: Apache/X.Y.ZZ (Ubuntu)
< Content-Type: text/html; charset=iso-8859-1
```

* Lines starting with `>` indicate the **request**
* Lines starting with `<` indicate the **response**
* The response body follows after the headers

For even more detail, you can use:

```
curl -vvv inlanefreight.com
```

---

## Browser Developer Tools (DevTools)

Modern browsers include **Developer Tools (DevTools)**, which are extremely valuable for analyzing web requests during assessments.

![devtools_network_requests](https://github.com/user-attachments/assets/b2f3106b-bdf0-4531-947a-be3466bcd155)

### Opening DevTools

* `CTRL + SHIFT + I`
* `F12`

### Network Tab

The **Network** tab displays all HTTP requests made by a page.

After opening the Network tab and refreshing the page, you can observe:

* Request method (GET, POST, etc.)
* Requested resource (URL and path)
* Response status codes
* Timing and size information

Filters can be used to quickly locate specific requests when many are present.
