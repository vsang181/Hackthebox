# HTML (HyperText Markup Language)

HTML is the **core building block of the front end** of every web application you will test. If you strip everything else away—CSS, JavaScript, frameworks—HTML is what remains. Every page you see in a browser is ultimately defined by HTML, and the browser’s job is simply to interpret it and render it for you.

As a tester, you should treat HTML as the **skeleton of the application**. It defines what exists on the page: text, forms, inputs, buttons, images, links, and structure. If something appears on the page, it is represented somewhere in HTML.

---

## A Minimal HTML Page

Here is a very basic HTML document:

```html
<!DOCTYPE html>
<html>
    <head>
        <title>Page Title</title>
    </head>
    <body>
        <h1>A Heading</h1>
        <p>A Paragraph</p>
    </body>
</html>
```

What this means for you:

* The browser reads the HTML from top to bottom
* It understands the structure, not the visual style
* It renders elements based on their meaning and position

The `<title>` appears in the browser tab, while the `<h1>` and `<p>` elements appear in the visible page area.

---

## HTML as a Tree (DOM Structure)

HTML is not flat. It is structured as a **tree**, very similar to XML.

Conceptually, the document above looks like this:

```
document
 └── html
     ├── head
     │   └── title
     └── body
         ├── h1
         └── p
```

Every element:

* Can contain other elements
* Has a parent and possibly children
* Exists at a specific position in the hierarchy

This hierarchical structure is critical for both **rendering** and **security testing**.

<img width="452" height="434" alt="Screenshot 2026-01-06 at 22 33 34" src="https://github.com/user-attachments/assets/94897900-d8e3-4645-84d0-befe981700d6" />

---

## HTML Elements and Tags

An HTML element consists of:

* An **opening tag**
* Optional **attributes**
* **Content**
* A **closing tag**

Example:

```html
<p id="para1" class="red-paragraph">
    A Paragraph
</p>
```

What you should notice:

* `<p>` defines the element type
* `id` and `class` are attributes
* The text is the element’s content

Attributes such as `id` and `class` are heavily used by:

* CSS (styling)
* JavaScript (logic)
* Security exploits (for example, DOM-based XSS)

Both the tags and the content together form the complete element.

---

## The Role of `<head>` and `<body>`

HTML pages are broadly split into two main sections:

### `<head>`

This section usually contains:

* Page metadata
* The page title
* CSS definitions (`<style>`)
* JavaScript includes (`<script>`)

Most of what is in `<head>` is **not directly visible** to the user.

---

### `<body>`

This is where the visible page lives:

* Text
* Forms
* Buttons
* Images
* Input fields
* Interactive elements

If a user can click it, type into it, or see it, it is almost certainly inside `<body>`.

---

## URL Encoding (Percent-Encoding)

One important concept you must understand early is **URL encoding**, also known as **percent-encoding**.

Browsers can only safely transmit a limited character set in URLs (ASCII). Characters outside this set must be encoded so they can be transmitted reliably.

The encoding format is:

```
% + two hexadecimal characters
```

Example:

* `'` (single quote) → `%27`
* Space → `%20` or `+`

Common encodings you will see constantly during testing:

| Character | Encoding |
| --------- | -------- |
| Space     | `%20`    |
| `!`       | `%21`    |
| `"`       | `%22`    |
| `#`       | `%23`    |
| `$`       | `%24`    |
| `%`       | `%25`    |
| `&`       | `%26`    |
| `'`       | `%27`    |
| `(`       | `%28`    |
| `)`       | `%29`    |

Why this matters to you:

* Payloads are often URL-encoded automatically
* Filters may decode input differently than you expect
* Double-encoding and partial encoding bugs are common
* Injection vulnerabilities often depend on encoding behaviour

Most modern tools (including Burp Suite) let you encode and decode values quickly. You should get used to switching between encoded and decoded views without thinking about it.

---

## The Document Object Model (DOM)

Once the browser parses HTML, it builds an internal representation called the **Document Object Model (DOM)**.

In simple terms:

* HTML becomes a live object tree
* JavaScript can read and modify it
* The page can change dynamically without reloading

The DOM is defined by the W3C as:

> “A platform- and language-neutral interface that allows programs and scripts to dynamically access and update the content, structure, and style of a document.”

The DOM is divided into:

* **Core DOM** – common to all document types
* **XML DOM** – specific to XML
* **HTML DOM** – specific to HTML

When you interact with elements programmatically, you are interacting with the DOM, not raw HTML.

Examples you will see:

* `document`
* `document.head`
* `document.body`
* `document.getElementById("id")`

---

## Why the DOM Matters for Security

Understanding the DOM helps you:

* Locate where elements actually live
* Identify client-side logic
* Understand how JavaScript manipulates the page
* Exploit front-end vulnerabilities such as **XSS**

Many vulnerabilities exist **only because developers trust the DOM too much**.

For example:

* User input written directly into the DOM
* JavaScript building HTML dynamically
* Hidden fields manipulated client-side
* DOM-based XSS that never touches the server

If you know how to navigate and reason about the DOM, front-end testing becomes far more systematic and far less guesswork.

---

## Key Takeaways

* HTML defines the structure of every web page
* Browsers interpret HTML and build a DOM tree
* HTML elements form a hierarchy, not a flat list
* Attributes like `id` and `class` are critical for interaction
* URL encoding affects how data is transmitted and parsed
* The DOM is live and can be manipulated at runtime
* Many front-end vulnerabilities rely on DOM behaviour

If you get comfortable reading HTML and visualising the DOM tree in your head, you will be able to **spot client-side weaknesses quickly and reliably**, especially when combined with JavaScript analysis in the next section.
