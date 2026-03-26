# Login Form Brute Forcing with Hydra

Web login forms require more preparation than Basic Auth because Hydra needs to know the form structure before it can construct valid requests. The reconnaissance step is not optional: without accurate parameter names and a reliable failure condition, every attempt will either fail silently or produce false positives.

***

## The Three-Step Reconnaissance Process

Before writing the Hydra command, you need three pieces of information from the target form.

**Step 1: Inspect the HTML source**

Right-click the page, select Inspect, and find the `<form>` element. You need:
- The `action` attribute (the submission path, e.g., `/login` or `/`)
- The `method` attribute (should be `POST`)
- The `name` attribute of each input field

```html
<form method="POST">
    <input type="text"     name="username">
    <input type="password" name="password">
    <input type="submit"   value="Login">
</form>
```

From this: path is `/`, fields are `username` and `password`.

**Step 2: Capture a live request via Network tab or Burp Suite**

Open DevTools (F12), go to the Network tab, submit any test credentials, and find the POST request. This confirms the exact parameter names and the submission path without relying solely on HTML inspection, which can be misleading if JavaScript modifies the form on submission.

**Step 3: Identify the failure or success condition**

Submit wrong credentials and note the exact error message. This string becomes your `F=` condition. Common examples:

| Application Response | Hydra Condition |
|--------------------|----------------|
| "Invalid credentials" | `F=Invalid credentials` |
| "Wrong password" | `F=Wrong password` |
| "Login failed" | `F=Login failed` |
| 302 redirect on success | `S=302` |
| "Dashboard" appears on success | `S=Dashboard` |

***

## Building the params String

The params string is the most critical and error-prone part of a form brute force. It follows a strict three-part colon-separated format:

```
"path:POST_body:condition"
```

Applied to the example:

```
"/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

- `/` is the form action path
- `username=^USER^&password=^PASS^` mirrors the exact POST body the browser sends, with `^USER^` and `^PASS^` as Hydra's placeholders
- `F=Invalid credentials` is the exact failure string from the server response

If the form submits to `/login` instead of `/`, or the field is called `user` instead of `username`, the attack will silently fail on every attempt. Accuracy here is not optional.

***

## The Full Hydra Command

```bash
# Download wordlists
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt

# Run the attack
hydra -L top-usernames-shortlist.txt \
      -P 2023-200_most_used_passwords.txt \
      -f \
      TARGET_IP \
      -s PORT \
      http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

With 17 usernames and 200 passwords, Hydra runs 3,400 total attempts (17 x 200). At 16 parallel threads, this completes in seconds against a local target with no rate limiting.

***

## Handling CSRF Tokens

Modern login forms often include a hidden CSRF token field that changes with every page load. A static params string will fail because the token in your request will not match the server's expected value. In this case, you need to capture the token dynamically before each attempt.

For simple cases, Burp Suite's Intruder with a macro handles this automatically. For script-based approaches:

```python
import requests
from bs4 import BeautifulSoup

session = requests.Session()

# Step 1: GET the login page to capture the CSRF token
response = session.get("http://TARGET/login")
soup = BeautifulSoup(response.text, 'html.parser')
csrf_token = soup.find('input', {'name': 'csrf_token'})['value']

# Step 2: POST with the token included
response = session.post("http://TARGET/login", data={
    'username': 'admin',
    'password': 'password123',
    'csrf_token': csrf_token
})
```

Hydra does not natively handle rotating CSRF tokens, which is one of its primary limitations against modern web applications. This is where custom Python scripts or Burp Suite's Intruder become necessary.

***

## Common Reasons a Form Attack Fails

| Symptom | Likely Cause |
|--------|-------------|
| All attempts flagged as success | Wrong failure condition string, or typo in it |
| All attempts flagged as failure | Wrong parameter names or form path |
| Zero results, no errors | Form requires HTTPS and HTTP was used |
| Very slow with many errors | Rate limiting or CAPTCHA active |
| Hydra reports success but creds don't work | Success condition matched unrelated page content |

The most common mistake by far is a slightly wrong failure string. If the server returns "Invalid credentials!" (with an exclamation mark) and your condition is `F=Invalid credentials` (without), every response looks like a success to Hydra. Always copy the exact failure string directly from the browser's response body.
