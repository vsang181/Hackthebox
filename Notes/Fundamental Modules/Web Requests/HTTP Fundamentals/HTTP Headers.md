# HTTP Headers

We have seen examples of HTTP request and response headers in the previous section. HTTP headers pass information between the client and the server. Some headers are used only in requests or responses, while others are common to both.

Headers can have one or multiple values, appended after the header name and separated by a colon (`:`). HTTP headers can be divided into the following categories:

* General Headers
* Entity Headers
* Request Headers
* Response Headers
* Security Headers

Let us discuss each of these categories.

---

## General Headers

General headers are used in both HTTP requests and responses. They describe the message itself rather than its content.

| Header     | Example                               | Description                                                                                                                        |
| ---------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| Date       | `Date: Wed, 16 Feb 2022 10:38:44 GMT` | Holds the date and time at which the message originated. It is preferred to use the standard UTC time zone.                        |
| Connection | `Connection: close`                   | Dictates whether the network connection should remain open after the request finishes. Common values are `close` and `keep-alive`. |

---

## Entity Headers

Entity headers can be present in both requests and responses. They describe the content (entity) being transferred and are commonly seen in responses and POST or PUT requests.

| Header           | Example                       | Description                                                                                                         |
| ---------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Content-Type     | `Content-Type: text/html`     | Describes the type of resource being transferred. The charset parameter denotes the encoding standard (e.g. UTF-8). |
| Media-Type       | `Media-Type: application/pdf` | Similar to Content-Type and describes the media format of the transferred data.                                     |
| Boundary         | `boundary="b4e4fbd93540"`     | Acts as a delimiter to separate multiple parts of a multipart message, such as form data.                           |
| Content-Length   | `Content-Length: 385`         | Specifies the size of the message body in bytes.                                                                    |
| Content-Encoding | `Content-Encoding: gzip`      | Indicates transformations applied to the data, such as compression.                                                 |

---

## Request Headers

Request headers are sent by the client and do not directly relate to the content of the message body.

| Header        | Example                                  | Description                                                                                        |
| ------------- | ---------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Host          | `Host: www.inlanefreight.com`            | Specifies the host being requested. Important for virtual host enumeration.                        |
| User-Agent    | `User-Agent: curl/7.77.0`                | Identifies the client software, browser, and operating system.                                     |
| Referer       | `Referer: http://www.inlanefreight.com/` | Indicates the origin of the request. This header can be spoofed and should not be trusted blindly. |
| Accept        | `Accept: */*`                            | Defines which media types the client can process.                                                  |
| Cookie        | `Cookie: PHPSESSID=b4e4fbd93540`         | Contains key–value pairs used for session management and tracking.                                 |
| Authorization | `Authorization: BASIC cGFzc3dvcmQK`      | Used for authentication via credentials or tokens.                                                 |

A complete list of request headers and their usage can be found in official HTTP documentation.

---

## Response Headers

Response headers provide additional context about the server and the response itself.

| Header           | Example                                     | Description                                                          |
| ---------------- | ------------------------------------------- | -------------------------------------------------------------------- |
| Server           | `Server: Apache/2.2.14 (Win32)`             | Reveals information about the web server software and version.       |
| Set-Cookie       | `Set-Cookie: PHPSESSID=b4e4fbd93540`        | Sends cookies to the client for storage and reuse.                   |
| WWW-Authenticate | `WWW-Authenticate: BASIC realm="localhost"` | Indicates the authentication scheme required to access the resource. |

---

## Security Headers

Security headers enforce browser-side security policies to protect against common web attacks.

| Header                    | Example                                       | Description                                                                                         |
| ------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Content-Security-Policy   | `Content-Security-Policy: script-src 'self'`  | Restricts the sources from which scripts and other resources may be loaded, mitigating XSS attacks. |
| Strict-Transport-Security | `Strict-Transport-Security: max-age=31536000` | Forces HTTPS usage and prevents protocol downgrade attacks.                                         |
| Referrer-Policy           | `Referrer-Policy: origin`                     | Controls how much referrer information is included in requests.                                     |

> **Note:** This section lists only a subset of commonly used HTTP headers. Applications may also define custom headers. A full list of standard HTTP headers is available in official specifications.

---

## Using cURL to Inspect Headers

To display **only response headers**, use the `-I` flag (HEAD request):

```bash
curl -I https://www.inlanefreight.com
```

To display **both headers and response body**, use the `-i` flag:

```bash
curl -i https://www.inlanefreight.com
```

To modify request headers, use the `-H` flag. For example, to set a custom User-Agent:

```bash
curl https://www.inlanefreight.com -A 'Mozilla/5.0'
```

---

## Browser DevTools

HTTP headers can also be inspected using browser developer tools:

![devtools_network_requests_details](https://github.com/user-attachments/assets/8d452c53-31c8-4174-8a3c-a28d9a578987)

1. Open DevTools (`F12` or `CTRL + SHIFT + I`)
2. Navigate to the **Network** tab
3. Reload the page
4. Click on any request
5. View **Request Headers** and **Response Headers** under the **Headers** section
6. Use the **Raw** view to see headers in their original format
7. Check the **Cookies** tab to inspect cookies used by the request
