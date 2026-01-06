# Cascading Style Sheets (CSS)

CSS (Cascading Style Sheets) is the language you use to **control how HTML looks**. While HTML defines *what exists* on a page, CSS defines *how it is presented*. Every colour, font, layout choice, animation, and visual effect you see in a web application is influenced—directly or indirectly—by CSS.

As you analyse web applications, you should think of CSS as the **visual logic layer** of the front end. It does not usually enforce security rules, but it often hides, reveals, or rearranges elements in ways that directly affect how vulnerabilities can be exploited.

---

## What CSS Does (At a Practical Level)

CSS allows you to define how HTML elements are rendered in the browser. You can control, for example:

* Fonts and text size
* Colours and backgrounds
* Alignment and spacing
* Positioning and layout
* Visibility and layering
* Transitions and animations

A simple CSS example looks like this:

```css
body {
  background-color: black;
}

h1 {
  color: white;
  text-align: center;
}

p {
  font-family: helvetica;
  font-size: 10px;
}
```

What this means in practice:

* Every `<body>` element will have a black background
* All `<h1>` elements will appear white and centred
* All `<p>` elements will use Helvetica at 10px size

Once defined, these rules apply automatically to every matching element on the page.

---

## Why IDs and Classes Matter

You will often see HTML elements with attributes like `id` and `class`. These are not decoration—they exist so that CSS (and JavaScript) can target specific elements.

Example:

```html
<p id="warning" class="highlight">
    Access denied
</p>
```

This allows you to:

* Style **one specific element** using its `id`
* Style **groups of elements** using a shared `class`

From a testing perspective, this matters because:

* Elements may be hidden using CSS instead of proper access control
* UI restrictions may rely purely on styling
* Sensitive functionality may exist in the DOM but be visually concealed

CSS can *hide*, but it cannot *protect*.

---

## CSS Syntax Basics

CSS follows a very consistent structure:

```css
selector {
  property: value;
}
```

Where:

* The **selector** identifies which elements are affected
* The **property** defines what you are changing
* The **value** defines how it changes

Example:

```css
button {
  display: none;
}
```

This hides all buttons visually, but they may still:

* Exist in the DOM
* Be reachable via JavaScript
* Be triggered manually

This distinction is critical during front-end security testing.

---

## Advanced CSS Capabilities

CSS is far more powerful than many people realise. Modern CSS supports:

* Transitions and animations
* Responsive layouts
* Complex positioning systems
* Hardware-accelerated rendering
* 3D transformations

Animations can be defined using constructs such as:

* `@keyframes`
* `animation`
* `animation-duration`
* `animation-direction`

While visually impressive, these features can also:

* Obscure application behaviour
* Distract users from unexpected changes
* Hide malicious DOM manipulation

---

## CSS and JavaScript Together

CSS is often tightly coupled with JavaScript. JavaScript can:

* Add or remove CSS classes dynamically
* Modify styles at runtime
* Trigger animations based on user behaviour
* React to mouse movement or keystrokes

This combination enables:

* Dynamic interfaces
* Rich user experiences
* Single-page applications (SPAs)

It also enables:

* DOM-based vulnerabilities
* UI redressing
* Logic hidden behind visual states

You should always check whether **security decisions depend on CSS state**.

---

## CSS Beyond HTML

CSS is not limited to HTML pages. You will also encounter it in:

* XML-based layouts
* SVG graphics
* Mobile application UI frameworks
* Embedded interfaces

This means CSS knowledge transfers across platforms, not just websites.

---

## CSS Frameworks

Many developers avoid writing CSS from scratch and instead rely on **CSS frameworks**. These provide pre-built styles, layouts, and components.

Common examples you will encounter:

* **Bootstrap**
* **SASS**
* **Foundation**
* **Bulma**
* **Pure**

Frameworks:

* Speed up development
* Standardise UI behaviour
* Encourage reusable components

From a security perspective, they also:

* Create predictable DOM structures
* Introduce shared attack patterns
* May include outdated or misused components

Knowing common frameworks helps you recognise layout patterns quickly and focus your testing more efficiently.

---

## Why CSS Matters to You as a Tester

Even though CSS does not usually enforce security rules, it strongly influences:

* What users can see
* What users believe they can do
* Which elements appear accessible

You should never assume that:

* Hidden elements are inaccessible
* Disabled buttons cannot be triggered
* Visual restrictions equal real restrictions

CSS controls **appearance**, not **authority**.
