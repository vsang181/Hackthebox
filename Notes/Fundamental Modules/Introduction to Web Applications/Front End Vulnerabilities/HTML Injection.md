# HTML Injection

One of the most important front-end security concepts you need to internalise early is **input handling**. Any time an application accepts user input and reflects it back into the page, you should immediately ask yourself:

> Where is this input processed, and how is it rendered?

In many applications, validation and sanitisation are handled on the **back end**. However, there are plenty of cases where user input:

* Never reaches the server, or
* Is processed and rendered entirely on the **front end**

When that happens, **front-end validation becomes just as critical** as back-end validation. Relying on one without the other is a common and dangerous mistake.

---

## What HTML Injection Is

HTML Injection happens when **unsanitised user input is inserted directly into the page as HTML**.

This usually occurs in one of two ways:

* Previously submitted input is retrieved (for example, comments stored in a database)
* User input is immediately written into the page using JavaScript

If you control how your input is rendered, you can inject **arbitrary HTML**. The browser will treat it as legitimate page content.

This may not sound severe at first, but HTML injection can be used to:

* Insert fake login forms
* Display misleading content
* Embed external resources
* Manipulate user trust
* Prepare the ground for more serious attacks such as XSS

---

## Why HTML Injection Is Dangerous

When HTML injection is possible, an attacker can:

* Trick users into entering credentials into attacker-controlled forms
* Redirect users visually without changing the URL
* Deface the application interface
* Inject misleading messages or advertisements

Even without JavaScript execution, this can cause:

* Credential theft
* Loss of user trust
* Reputational damage
* Compliance issues

Web page defacement is a classic example. By injecting new HTML, the attacker can:

* Change page layout
* Replace branding
* Insert offensive or malicious content
* Make the application appear compromised

---

## A Simple Vulnerable Example

Consider a page with a single button that asks for your name and then displays it back to you.

From a user’s point of view:

* You click a button
* You enter your name
* The page shows: *“Your name is …”*

From a tester’s point of view, this is a **reflection point**.

Here is a simplified version of the page source:

```html
<!DOCTYPE html>
<html>
<body>

<button onclick="inputFunction()">Click to enter your name</button>
<p id="output"></p>

<script>
    function inputFunction() {
        var input = prompt("Please enter your name", "");

        if (input != null) {
            document.getElementById("output").innerHTML =
                "Your name is " + input;
        }
    }
</script>

</body>
</html>
```

The problem is subtle but critical:

```javascript
innerHTML = "Your name is " + input;
```

Your input is written **directly into the DOM as HTML**, without any filtering or sanitisation.

---

## Testing for HTML Injection

Testing for HTML injection is straightforward.

Instead of entering a normal name, you enter **HTML code** and observe how the page reacts.

For example, you might input:

```html
<style>
  body {
    background-image: url('https://example.com/image.svg');
  }
</style>
```

If the page immediately changes appearance, you have confirmed:

* Your input is treated as HTML
* No sanitisation is in place
* HTML injection is possible

In this specific example, refreshing the page resets everything because the logic is purely client-side. However, the vulnerability still exists and can often be far more dangerous when combined with persistence or server-side storage.

---

## HTML Injection vs XSS

HTML Injection and XSS are closely related, but they are **not the same thing**.

* **HTML Injection**
  You inject markup that the browser renders

* **Cross-Site Scripting (XSS)**
  You inject JavaScript that the browser executes

HTML injection is often:

* Easier to achieve
* Overlooked during testing
* A stepping stone toward XSS

If you can inject HTML, you should always test whether JavaScript execution is also possible.

---

## Why Front-End Only Issues Still Matter

It is easy to dismiss front-end-only vulnerabilities because:

* They reset on refresh
* They do not directly affect the database
* They do not persist by default

This is a mistake.

Front-end issues:

* Target **users**, not servers
* Can be chained with other vulnerabilities
* Are especially dangerous when admins are affected
* Often indicate deeper input-handling problems

---

## How This Should Shape Your Testing Mindset

Whenever you see user input reflected in the page:

* Assume it is unsafe until proven otherwise
* Look for `innerHTML`, `document.write`, or dynamic DOM creation
* Test with basic HTML payloads first
* Escalate to JavaScript payloads where possible
