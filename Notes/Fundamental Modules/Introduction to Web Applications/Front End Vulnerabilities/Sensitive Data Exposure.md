# Sensitive Data Exposure

All front-end components you have looked at so far execute **entirely on the client side**. Because of this, vulnerabilities in the front end do not usually cause direct, permanent damage to the back end by themselves. However, that does **not** make them harmless. Front-end weaknesses primarily put **users** at risk, and when those users include administrators, the impact can quickly escalate to full application or server compromise 

As a tester, you should never ignore front-end issues. They are often the **entry point** that enables access to sensitive functionality, privileged interfaces, or internal systems.

---

## What Sensitive Data Exposure Means

Sensitive Data Exposure refers to situations where **confidential or valuable information is visible to the client** in clear text. This information is usually found in:

* HTML page source
* Inline JavaScript
* External JavaScript files
* Comments left by developers
* Hidden form fields
* Embedded configuration values

This is **front-end source code**, not server-side code. Even though it is not meant to be “secret”, it is frequently treated as such by developers—and that is where the problem begins.

---

## Viewing Page Source

You can inspect a web page’s source code in several ways:

* Right-click → **View Page Source**
* Keyboard shortcut: **Ctrl + U**
* Through a web proxy such as **Burp Suite**

Even if right-click is disabled, source code is **never hidden**. Client-side code must be delivered to the browser in order to run.

For example, viewing the source of a public website shows:

* HTML structure
* Inline JavaScript
* External script references
* Embedded comments

You should always assume that **anything sent to the browser is readable by an attacker**.

---

## Commonly Exposed Sensitive Information

When reviewing page source, you should actively look for:

* Hard-coded credentials
* API keys or tokens
* Password hashes
* Test or debug accounts
* Hidden URLs or directories
* Internal hostnames or IPs
* Feature flags
* Debug parameters
* User-related data

This information is often left behind accidentally during development or testing.

---

## A Simple but Dangerous Example

At first glance, the following login form looks completely normal:

```html
<form action="action_page.php" method="post">
    <div class="container">
        <label for="uname"><b>Username</b></label>
        <input type="text" required>

        <label for="psw"><b>Password</b></label>
        <input type="password" required>

        <!-- TODO: remove test credentials test:test -->

        <button type="submit">Login</button>
    </div>
</form>
```

The issue is not the form itself—it is the **developer comment**:

```html
<!-- TODO: remove test credentials test:test -->
```

This comment suggests:

* Test credentials exist
* They may still be valid
* They were never removed before deployment

Even if such findings are not common, they are **absolutely real**, and they do appear during assessments.

---

## Why This Matters in Practice

Information exposed on the front end can often be chained with other weaknesses to achieve much more serious outcomes, such as:

* Accessing admin panels
* Bypassing authentication
* Enumerating application functionality
* Targeting back end APIs
* Compromising supporting infrastructure (web servers, databases)

This is why reviewing page source is one of the **first steps** you should take when assessing any web application. It is low effort and frequently yields high-value results.

---

## Automation vs Manual Review

There are automated tools that can:

* Crawl page source
* Extract comments
* Identify hidden paths
* Flag suspicious strings

However, **manual review is still critical**. Context matters, and automated tools cannot always determine whether exposed data is actually sensitive or exploitable.

You should train yourself to:

* Skim source code efficiently
* Recognise red flags instantly
* Think about how exposed data could be abused

---

## Prevention and Defensive Considerations

From a defensive perspective, front-end source code should:

* Contain only what is strictly necessary
* Avoid leaving comments meant for developers
* Never include credentials or secrets
* Avoid embedding internal logic assumptions
* Be reviewed before deployment

Additional hardening steps include:

* Removing unused code paths
* Auditing client-side data exposure
* Classifying what data is allowed client-side
* Minimising information disclosure

Some teams also use JavaScript minification or obfuscation. While this may reduce casual inspection, it **does not provide real security** and should never be relied upon as a primary control.
