# Introduction to Cross-Site Scripting (XSS)

XSS is one of the most widespread and long-standing vulnerability classes in web security. It occurs when a web application includes unsanitised user input in its HTML output, causing the victim's browser to execute attacker-controlled JavaScript as if it were legitimate page code.

***

## How XSS Differs from Server-Side Vulnerabilities

XSS executes entirely on the client side. The back-end server is not directly compromised; instead, the attacker uses the trusted web application as a delivery mechanism to push malicious code to other users' browsers. This distinction matters for risk calculation:

- **Impact**: Medium at the individual user level (session theft, credential harvesting, browser exploitation)
- **Probability**: High, XSS remains one of the most commonly found vulnerabilities across all web applications
- **Combined risk**: Medium overall, but scales dramatically with the number of users affected

XSS is constrained by the browser's JavaScript engine and the same-origin policy in modern browsers. It cannot natively execute system-level code. However, when chained with a browser binary vulnerability (heap overflow, use-after-free), an XSS payload becomes the delivery mechanism for a full sandbox escape and OS-level code execution.

***

## Real-World Impact: Historical Examples

| Year | Incident | Impact |
|------|---------|--------|
| 2005 | Samy Worm on MySpace | 1 million users affected in under 24 hours via stored XSS self-propagation |
| 2014 | TweetDeck XSS | Self-retweeting tweet hit 38,000+ retweets in two minutes, forced platform shutdown |
| 2019 | Google Search XML library | Mutation XSS in one of the world's highest-traffic pages |
| 2010 | Apache Server XSS | Actively exploited to steal corporate user passwords |

The Samy Worm is the canonical demonstration of stored XSS at scale. The payload did three things: displayed "Samy is my hero" on a victim's profile, added Samy as a friend, and copied itself to the victim's profile to repeat the process for every subsequent viewer. A weaponised version of the same worm structure could have exfiltrated session cookies for every infected account.

***

## The Three XSS Types

### Stored (Persistent) XSS

The payload is written to the back-end database and served to every user who loads the affected page. No attacker interaction is required after the initial injection. This is the most severe type because one successful injection affects all future visitors.

**Typical injection points**: Comment fields, user profile bios, forum posts, product reviews, chat messages

### Reflected (Non-Persistent) XSS

The payload travels in the HTTP request (typically a URL parameter) and is immediately included in the response without being stored. The server processes the input and reflects it back in the HTML. The attacker must convince each victim to click a crafted link containing the payload.

**Typical injection points**: Search boxes, error messages, login redirect URLs, query string parameters

```
http://target.com/search?q=<script>alert(document.cookie)</script>
```

### DOM-Based XSS

The vulnerability lives entirely in client-side JavaScript. The server never sees the malicious input. The page's own scripts read from sources like `window.location.hash`, `document.URL`, or `document.referrer` and write that data into the DOM without sanitisation.

**Typical sources**: URL fragments (`#`), `localStorage`, `postMessage` handlers
**Typical sinks**: `innerHTML`, `document.write()`, `eval()`, `setTimeout()` with string argument

***

## XSS Attack Capabilities

Once JavaScript executes in a victim's browser under the application's origin, the attacker can:

- Steal session cookies via `document.cookie` and replay them to hijack the account
- Make authenticated API calls on the victim's behalf (change email, password, or payment details)
- Inject a fake login overlay to capture plaintext credentials
- Log keystrokes across the entire page
- Access HTML5 APIs including geolocation, camera, and microphone (with browser permission prompts)
- Redirect the victim to attacker-controlled phishing infrastructure
- Deliver drive-by malware downloads
- Use BeEF (Browser Exploitation Framework) to hook the browser for persistent command and control
- Mine cryptocurrency using the victim's CPU via JavaScript

***

## XSS vs Other Web Vulnerabilities

XSS has a deceptively low individual impact compared to SQLi or RCE, which is why it is sometimes undervalued during assessments. The correct way to assess it is to multiply the per-user impact by the number of users exposed. A stored XSS on a banking application's statement page does not just affect one user: it affects every account holder who views their statement until the vulnerability is patched.

The module ahead covers all three XSS types in depth, from identifying injection points and crafting payloads, through to real exploitation techniques including session hijacking, phishing via injected content, blind XSS for targeting admin panels, and DOM-based source and sink analysis.
