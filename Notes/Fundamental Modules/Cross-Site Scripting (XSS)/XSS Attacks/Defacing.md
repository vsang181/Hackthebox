# XSS-Based Website Defacing

Defacing via stored XSS is one of the most visible and impactful ways to abuse a persistent injection point. Because the payload is stored server-side, every visitor sees the modified page without needing to receive a crafted link.

***

## Why Stored XSS Is Used for Defacing

Stored XSS is the only XSS type that affects all visitors automatically. Reflected and DOM-based variants require each victim to visit a crafted URL, which would defeat the purpose of a public-facing defacement. The stored payload fires for every page load until someone with backend access removes it from the database.

The reputational damage is often greater than the technical damage. When hacker groups defaced NHS websites in 2018, the media coverage caused public concern that went far beyond the actual technical compromise involved.

***

## The Four Core Defacement Properties

| Property | JavaScript | Effect |
|----------|-----------|--------|
| Background colour | `document.body.style.background = "#141d2b"` | Changes page background to a hex colour |
| Background image | `document.body.background = "https://example.com/img.svg"` | Sets a full image as background |
| Page title | `document.title = 'Hacked'` | Changes the browser tab/window title |
| Body content | `document.getElementsByTagName('body')[0].innerHTML = '...'` | Replaces entire visible page content |

***

## Building the Defacement Payload

Each property is injected as a separate stored XSS entry, or combined into one. The full-body replacement is the most impactful since it wipes all visible content and replaces it with attacker-controlled HTML:

```html
<!-- Step 1: Set dark background -->
<script>document.body.style.background = "#141d2b"</script>

<!-- Step 2: Change browser tab title -->
<script>document.title = 'Hacked'</script>

<!-- Step 3: Replace all visible page content -->
<script>
document.getElementsByTagName('body')[0].innerHTML = '<center><h1 style="color:white">You Have Been Hacked</h1></center>'
</script>
```

Always minify the HTML to a single line before embedding it inside `innerHTML` to avoid quote conflicts and broken JavaScript syntax. Test the HTML locally first before committing it to the live injection.

***

## What the Page Source Looks Like After Injection

The original source code stays intact. Injected payloads append at the point of injection in the stored data, and they execute after the original markup loads:

```html
<div></div>
<ul id="todo">
  <ul><script>document.body.style.background = "#141d2b"</script></ul>
  <ul><script>document.title = 'Hacked'</script></ul>
  <ul><script>document.getElementsByTagName('body')[0].innerHTML = '...'</script></ul>
</ul>
```

To a site administrator inspecting the raw source, the original code is still visible. To any ordinary visitor, the page looks completely replaced. The `innerHTML` payload effectively hides all original content by overwriting the rendered DOM, even though the underlying HTML file is untouched.

***

## Practical Considerations

A few things are worth accounting for when constructing a defacement payload in a real assessment:

- **Injection position matters**: If your stored payload appears in the middle of the page rather than the end, elements injected after it by the server may render on top of your defaced content. In that case, `innerHTML` on the body is more reliable than background changes alone since it overwrites everything
- **Quote escaping**: Embedding HTML inside a JavaScript string means you must alternate quote types or use HTML entities. Single quotes in the outer script block, double quotes inside the HTML string is the standard approach
- **Persistence resistance**: Using `innerHTML` to replace the body makes it harder for administrators to quickly identify and revert the change by visually inspecting the rendered page, since the original content is no longer visible without viewing raw source
- **Avoiding overwrites**: In a real defacement, submitting the `innerHTML` payload last ensures it overwrites any partial changes made by earlier payloads in the same session
