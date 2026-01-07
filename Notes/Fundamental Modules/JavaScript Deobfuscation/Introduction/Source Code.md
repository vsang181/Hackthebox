# Source Code

Most websites nowadays utilise JavaScript to perform their functionality. HTML is used to define the website’s structure and elements, while CSS is responsible for its visual design. JavaScript is used to implement the logic and behaviour required for the website to function. This logic runs in the background, while users interact only with the visible front end.

Although all of this source code is available on the client side, it is rendered by the browser, so it often goes unnoticed. However, when attempting to understand a page’s client-side behaviour, the first step is usually to inspect the page’s source code. This section demonstrates how to uncover this source code and understand its general purpose.

# HTML

We begin the exercise by opening Firefox in the PwnBox and visiting the URL provided in the question:
Secret Serial Generator: This page generates secret serials.

The page displays the text Secret Serial Generator, without any input fields or obvious functionality. The next step is to inspect the source code. This can be done by pressing CTRL + U, which opens the source view of the website:
HTML code snippet for a webpage titled Secret Serial Generator with CSS for full-page width and height.

This view allows us to examine the HTML source code of the website.

HTML source code can contain various forms of information, including comments, which may assist in understanding how the page works. In some cases, developers leave sensitive information within these comments, which can be exploited. For this reason, HTML comments should always be reviewed carefully.

# CSS

CSS can be defined internally within the same HTML file using <style> elements, or externally in a separate .css file that is referenced from the HTML code.

In this case, the CSS is defined internally, as shown in the following snippet:
Code: html

```
<style>
    *,
    html {
        margin: 0;
        padding: 0;
        border: 0;
    }
    ...SNIP...
    h1 {
        font-size: 144px;
    }
    p {
        font-size: 64px;
    }
</style>
```

If a page’s CSS is defined externally, the .css file is referenced using a <link> tag within the HTML head, as shown below:
Code: html

```
<head>
    <link rel="stylesheet" href="style.css">
</head>
```

# JavaScript

The same principle applies to JavaScript. It can be written internally between <script> elements or placed in a separate .js file that is referenced from the HTML code.

In the HTML source, the JavaScript file is referenced externally:
Code: html

```
<script src="secret.js"></script>
```

We can inspect the script by opening secret.js, which loads the contents of the file. The code appears highly complex and difficult to understand:
Code: javascript

```
eval(function (p, a, c, k, e, d) { e = function (c) { '...SNIP... |true|function'.split('|'), 0, {}))
```

This complexity is the result of code obfuscation. What is it? How is it implemented? Where is it commonly used?
