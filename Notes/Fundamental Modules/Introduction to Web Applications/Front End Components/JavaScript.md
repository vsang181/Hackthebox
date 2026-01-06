# JavaScript

JavaScript is one of the most widely used programming languages in the world and a **cornerstone of modern web applications**. While it is most commonly associated with front-end development and browser-based execution, JavaScript is not limited to the client side. With environments such as Node.js, it is also used to build full back end applications.

As you have already seen, **HTML defines structure** and **CSS defines appearance**. JavaScript is what brings everything to life. Without it, most web pages would be static documents with little to no interactivity.

---

## What JavaScript Does on the Front End

On the client side, JavaScript is responsible for:

* Handling user interactions (clicks, input, gestures)
* Updating page content dynamically
* Validating user input
* Sending and receiving data from the back end
* Controlling application logic in real time

Whenever a page updates without refreshing, reacts instantly to your actions, or behaves like a desktop application, JavaScript is doing the heavy lifting.

---

## Loading JavaScript in a Web Page

JavaScript is included in HTML using the `<script>` tag.

### Inline JavaScript

```html
<script type="text/javascript">
    // JavaScript code goes here
</script>
```

This approach embeds JavaScript directly into the page. You will often see this in smaller applications or legacy code.

---

### External JavaScript Files

```html
<script src="./script.js"></script>
```

This is the preferred approach for most applications:

* Cleaner HTML
* Easier maintenance
* Better performance through caching

As a tester, external scripts are especially interesting because they often contain **business logic, API calls, and hidden functionality**.

---

## A Simple JavaScript Example

```javascript
document.getElementById("button1").innerHTML = "Changed Text!";
```

What this does:

* Locates an element with the ID `button1`
* Replaces its content dynamically

This type of DOM manipulation is extremely common. It is also a common source of **DOM-based vulnerabilities**, especially when user input is involved.

---

## JavaScript and the DOM

JavaScript does not usually modify raw HTML files. Instead, it interacts with the **Document Object Model (DOM)** created by the browser.

This allows JavaScript to:

* Create new elements
* Modify existing elements
* Remove elements
* Change attributes and styles
* React to events

From a security perspective, you should always be asking:

* Where does the data come from?
* Is user input written into the DOM?
* Is it sanitised before being used?

---

## How JavaScript Is Used in Modern Web Applications

Most modern applications rely heavily on JavaScript for:

* Real-time content updates
* Client-side routing
* Form handling and validation
* State management
* API communication

JavaScript can send HTTP requests to the back end using technologies such as **AJAX** and `fetch`, allowing the page to exchange data without reloading.

This is the foundation of:

* Single Page Applications (SPAs)
* Rich client-side interfaces
* API-driven architectures

---

## JavaScript and Performance

All modern browsers include highly optimised JavaScript engines that execute code locally on your machine. This allows:

* Fast execution
* Reduced server load
* Immediate feedback to users

However, it also means:

* Logic runs in an environment fully controlled by the user
* Any client-side checks can be bypassed
* Security decisions **must never rely on JavaScript alone**

---

## JavaScript and CSS Together

JavaScript is often used alongside CSS to:

* Toggle visibility
* Trigger animations
* Apply dynamic styles
* Respond to mouse movement or keystrokes

This combination enables visually impressive interfaces, but it also introduces risks:

* Hidden elements may still be accessible
* UI-based restrictions may be cosmetic only
* Application logic may be tied to visual state

Always remember: **appearance is not authority**.

---

## JavaScript Frameworks

As applications grow, writing everything in “vanilla” JavaScript becomes inefficient. This is why frameworks and libraries are widely used.

These frameworks:

* Abstract common functionality
* Provide reusable components
* Enable dynamic HTML generation
* Improve development speed

Common front-end JavaScript frameworks you will encounter:

* **Angular**
* **React**
* **Vue**
* **jQuery**

Some frameworks use JavaScript directly, while others use higher-level languages that compile down to JavaScript.

From a testing perspective, frameworks:

* Create predictable patterns
* Hide complexity behind abstractions
* Often introduce framework-specific vulnerabilities

Learning to recognise framework structures will save you a significant amount of time.

---

## Why JavaScript Matters for Security Testing

JavaScript often contains:

* Hidden API endpoints
* Hard-coded URLs
* Access control logic
* Feature toggles
* Debug functionality

Client-side logic is **never a security boundary**. Anything enforced only in JavaScript can usually be bypassed.

You should assume that:

* Users can read the JavaScript
* Users can modify execution
* Users can replay or forge requests
