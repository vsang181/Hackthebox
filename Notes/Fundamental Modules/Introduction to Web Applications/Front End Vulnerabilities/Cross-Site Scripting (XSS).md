# Cross-Site Scripting (XSS)

Once you understand **HTML Injection**, the next logical step is **Cross-Site Scripting (XSS)**. In practice, the two are closely related. If you can inject HTML, you should *always* test whether you can also inject **JavaScript**.

The key difference is simple:

* **HTML Injection** → you control how the page looks
* **XSS** → you control what the page *does*

With XSS, you are no longer limited to visual changes. You are executing code **inside the victim’s browser**, in the context of the vulnerable application.

---

## What XSS Actually Is

Cross-Site Scripting occurs when **untrusted input is executed as JavaScript** in the victim’s browser.

This means:

* The code runs with the same privileges as the application
* It can access cookies, DOM objects, and browser APIs
* It runs without the victim realising anything is wrong

Once you reach this point, you are no longer just testing the application — you are interacting directly with the user’s session.

---

## Why XSS Is Dangerous

If you can execute JavaScript in a user’s browser, you may be able to:

* Steal session cookies
* Hijack authenticated sessions
* Perform actions on behalf of the user
* Modify page content dynamically
* Log keystrokes
* Redirect users transparently
* Chain into further attacks

If the victim is an **administrator**, the impact can escalate extremely quickly.

---

## The Three Main Types of XSS

You will usually categorise XSS into one of the following types:

| Type              | Description                                                                                                             |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Reflected XSS** | User input is processed and immediately reflected back in the response (for example, search queries or error messages). |
| **Stored XSS**    | User input is saved on the server (for example, in comments or posts) and executed every time it is viewed.             |
| **DOM-based XSS** | User input is processed entirely in the browser and written into the DOM using JavaScript.                              |

From a testing perspective, **DOM XSS is especially common** in modern applications that rely heavily on client-side logic.

---

## From HTML Injection to XSS

In the earlier HTML Injection example, user input was written directly into the page using `innerHTML`, with **no sanitisation at all**.

That is a red flag.

If input is written into the DOM without filtering, you should immediately try to escalate from HTML injection to JavaScript execution.

---

## A Simple DOM XSS Payload

A commonly used test payload looks like this:

```javascript
#"><img src=/ onerror=alert(document.cookie)>
```

What this payload does:

* Breaks out of the existing HTML context
* Injects a new `<img>` element
* Triggers JavaScript execution via the `onerror` handler
* Reads `document.cookie`
* Displays it in an alert dialog

If this executes successfully, you have confirmed **DOM-based XSS**.

---

## What Just Happened

When the browser processes your input:

* Your payload becomes part of the DOM
* The browser treats it as legitimate page content
* The JavaScript executes automatically

This happens **entirely on the client side**, without touching the server.

The key takeaway is this:

> If user input reaches the DOM unsanitised, the browser will trust it.

---

## Real-World Impact

Showing an alert box is not the goal — it is just proof of execution.

In a real attack, the JavaScript would likely:

* Send the cookie value to an attacker-controlled server
* Exfiltrate sensitive data silently
* Perform authenticated actions in the background

Session cookies are particularly valuable. If session management is weak, a stolen cookie may be enough to **fully impersonate the victim**.

---

## Why XSS Is Often Underestimated

XSS is sometimes dismissed because:

* It “only affects users”
* It does not directly compromise the server
* It may not persist

This is a mistake.

XSS:

* Targets trust relationships
* Breaks same-origin assumptions
* Often leads to full account compromise
* Is frequently chained with other vulnerabilities

Many large-scale breaches have started with “just XSS”.

---

## How You Should Test for XSS

Whenever you see user input reflected anywhere:

* Test for HTML injection first
* Escalate to JavaScript payloads
* Inspect how the DOM is modified
* Look for `innerHTML`, `document.write`, and dynamic rendering
* Assume client-side validation can be bypassed

Do not rely on visual behaviour alone. Always inspect the DOM and network traffic.
