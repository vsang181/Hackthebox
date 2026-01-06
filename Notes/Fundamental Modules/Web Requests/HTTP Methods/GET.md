# GET

Whenever we visit any URL, browsers default to sending a **GET** request to retrieve the remote resources hosted at that URL. Once the browser receives the initial page, it may send additional requests using various HTTP methods. This behavior can be observed through the **Network** tab in browser developer tools.

> **Exercise**
> Pick any website and monitor the **Network** tab in the browser developer tools as you visit it.
> This technique helps you understand how a web application communicates with its back end and is essential for web application assessments and bug bounty testing.

---

## HTTP Basic Authentication

In the exercise at the end of this section, the server prompts for a username and password. Unlike traditional login forms that submit credentials via HTTP parameters (typically using POST requests), **HTTP Basic Authentication** is handled directly by the web server. It protects specific pages or directories without interacting with the web application logic.

In this case, the credentials are:

```
admin:admin
```

![http_auth_login](https://github.com/user-attachments/assets/7875bdce-87a7-4913-bbe1-9023a1854524)

Once valid credentials are provided, access to the page is granted.

![http_auth_index](https://github.com/user-attachments/assets/5dc6727d-2a8c-45d5-9631-73bc3902066e)

---

## Accessing Basic Auth with cURL

To test this using cURL and view the response headers, we use the `-i` flag:

```bash
curl -i http://<SERVER_IP>:<PORT>/
```

### Response (Unauthenticated)

```http
HTTP/1.1 401 Authorization Required
Date: Mon, 21 Feb 2022 13:11:46 GMT
Server: Apache/2.4.41 (Ubuntu)
Cache-Control: no-cache, must-revalidate, max-age=0
WWW-Authenticate: Basic realm="Access denied"
Content-Length: 13
Content-Type: text/html; charset=UTF-8

Access denied
```

The presence of the `WWW-Authenticate: Basic` header confirms that **Basic HTTP Authentication** is required.

---

## Authenticating with cURL

We can authenticate using the `-u` flag:

```bash
curl -u admin:admin http://<SERVER_IP>:<PORT>/
```

### Response (Authenticated)

```html
<!DOCTYPE html>
<html lang="en">
<head>
...SNIP...
```

---

## Credentials in the URL

Basic authentication credentials can also be supplied directly in the URL:

```bash
curl http://admin:admin@<SERVER_IP>:<PORT>/
```

This method also grants access, though it is discouraged in practice due to security risks.

> **Exercise**
> Add the `-i` flag to the authenticated request and compare the response headers to the unauthenticated response.

---

## HTTP Authorization Header

Using the `-v` flag allows us to inspect the full request and response:

```bash
curl -v http://admin:admin@<SERVER_IP>:<PORT>/
```

### Request Header (Excerpt)

```http
Authorization: Basic YWRtaW46YWRtaW4=
```

The value `YWRtaW46YWRtaW4=` is the Base64 encoding of:

```
admin:admin
```

> **Note**
> Modern authentication methods (e.g., JWT) use:
>
> ```
> Authorization: Bearer <token>
> ```

---

## Manually Setting the Authorization Header

We can manually supply the authorization header using `-H`:

```bash
curl -H 'Authorization: Basic YWRtaW46YWRtaW4=' http://<SERVER_IP>:<PORT>/
```

This also successfully authenticates the request.

---

## GET Parameters

Once authenticated, we gain access to a **City Search** feature. Entering a search term triggers a backend request.

Using browser developer tools, we can observe a request similar to:

```
search.php?search=le
```

This indicates the application uses **GET parameters**.

---

## Reproducing the Request with cURL

We can send the same request directly using cURL:

```bash
curl 'http://<SERVER_IP>:<PORT>/search.php?search=le' \
     -H 'Authorization: Basic YWRtaW46YWRtaW4='
```

### Response

```text
Leeds (UK)
Leicester (UK)
```

> **Note**
> When copying a request from browser developer tools, unnecessary headers can usually be removed.
> Authentication headers are typically the only required ones.

---

## Replaying Requests with Fetch

Browser developer tools also allow requests to be copied as **Fetch** calls. These can be pasted directly into the JavaScript console to replay the same request using the Fetch API.

This method is useful for:

* Testing client-side behavior
* Replaying authenticated requests
* Inspecting API responses interactively
