# POST

In the previous section, we saw how **GET** requests are used for actions such as searching and accessing pages. However, whenever web applications need to transfer files or move user parameters out of the URL, they use **POST** requests.

Unlike HTTP GET, which places user parameters within the URL, **HTTP POST places parameters inside the request body**. This provides several advantages:

## Advantages of POST Requests

* **Lack of Logging**
  POST requests may transfer large files (for example, file uploads). Logging this data in URLs would be inefficient and insecure.

* **Less Encoding Requirements**
  URLs must conform to strict character rules. POST request bodies can carry binary data, and only parameter separators require encoding.

* **Ability to Send More Data**
  URL length limits vary across browsers, servers, CDNs, and URL shorteners. In practice, URLs should remain under ~2,000 characters, making them unsuitable for large data transfers.

---

## Login Forms

In this exercise, the application uses a **PHP login form** instead of HTTP Basic Authentication.

![web_requests_post_login](https://github.com/user-attachments/assets/db48cc6b-44ad-41fd-b09a-1a307b9be600)

After logging in with valid credentials (`admin:admin`), access is granted and the search functionality becomes available.

![web_requests_login_search](https://github.com/user-attachments/assets/fa4dc6fa-bbd6-4dea-933c-24c290e04361)

If we clear the **Network** tab in the browser developer tools and log in again, we can observe a **POST request** being sent containing the credentials.

### POST Request Data

```text
username=admin&password=admin
```

<img width="1276" height="478" alt="image" src="https://github.com/user-attachments/assets/53d2ebc1-d370-4c05-a823-d34c9792f56d" />

---

## Authenticating with cURL

We can manually replicate this request using cURL.

### Sending a POST Request

```bash
curl -X POST -d 'username=admin&password=admin' http://<SERVER_IP>:<PORT>/
```

If authentication is successful, the returned HTML no longer contains the login form and instead shows the authenticated content (for example, the search interface).

> **Tip**
> Many login forms redirect users after authentication (for example, to `/dashboard.php`).
> Use the `-L` flag with cURL to follow redirects automatically.

---

## Authenticated Cookies

Upon successful authentication, the server issues a **session cookie** to maintain the logged-in state.

### Viewing the Cookie

```bash
curl -X POST -d 'username=admin&password=admin' http://<SERVER_IP>:<PORT>/ -i
```

Example response header:

```http
Set-Cookie: PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1; path=/
```

---

## Reusing the Session Cookie

Once authenticated, the cookie alone is sufficient to access protected resources.

### Using the Cookie with cURL

```bash
curl -b 'PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1' http://<SERVER_IP>:<PORT>/
```

Alternatively, the cookie can be supplied explicitly as a header:

```bash
curl -H 'Cookie: PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1' http://<SERVER_IP>:<PORT>/
```

> **Key Point**
> Many applications rely entirely on session cookies for authentication.
> Possession of a valid cookie may be sufficient to impersonate a user.

---

## Modifying Cookies in the Browser

If logged out, the application displays the login page again.
However, manually inserting a valid session cookie using browser developer tools immediately restores authenticated access.

This demonstrates why **session hijacking** and **XSS** vulnerabilities are so impactful.

---

## JSON Data in POST Requests

When interacting with the search functionality, the application sends a **POST request containing JSON data**.

### Example Request Body

```json
{"search":"london"}
```

This requires the request to specify the correct `Content-Type`.

### Relevant Headers

```http
Content-Type: application/json
Cookie: PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1
```

---

## Replicating the JSON Request with cURL

```bash
curl -X POST \
  -d '{"search":"london"}' \
  -b 'PHPSESSID=c1nsa6op7vtk7kdis7bcnbadf1' \
  -H 'Content-Type: application/json' \
  http://<SERVER_IP>:<PORT>/search.php
```

### Response

```json
["London (UK)"]
```

This allows direct interaction with backend endpoints without using the web interface.

---

## Why This Matters

Being able to manually craft POST requests is critical for:

* Web application penetration testing
* Bug bounty assessments
* API testing
* Automation and fuzzing

It is often **faster and more precise** than using a browser.

---

## Exercise

* Repeat the above request **without the cookie**
* Repeat it **without the Content-Type header**
* Observe how the application behaves differently

---

## Using Fetch for POST Requests

The same request can be reproduced using the browser’s **Fetch API** by copying it from the Network tab and executing it in the JavaScript console.

This method is useful for:

* Rapid testing
* Client-side analysis
* Understanding API behavior
