# ZAP Scanner

ZAP's built-in scanner provides the same three-stage workflow as Burp Scanner: spider, passive scan, and active scan. The key difference is that it is entirely free with no feature restrictions, making it a fully capable scanner for any budget.

***

## Spider

ZAP Spider maps the target by crawling all links it can find in the application. To start it, right-click a request in the History tab and select `Attack > Spider`, or use the HUD by clicking the Spider Start button (second button on the right pane) while on the target page.

If the target is not yet in scope, ZAP will prompt you to add it automatically before the scan begins. Once running, progress is visible both in the HUD spider button indicator and in the main ZAP UI under the Spider tab.

When the spider finishes, the Sites tab in the main UI displays a full expandable tree of all discovered URLs and sub-directories.

### Ajax Spider

ZAP also includes an Ajax Spider, launched from the third button on the right pane. The standard spider only follows static HTML links, while the Ajax Spider additionally detects links triggered by JavaScript and AJAX requests that fire after page load. Running Ajax Spider after the standard spider finishes gives more complete coverage, particularly for single-page applications or heavily JavaScript-driven sites. It takes longer but is worth running on modern web applications.

***

## Passive Scanner

ZAP runs its passive scanner automatically in the background as the spider makes requests. You do not need to trigger it manually. By the time the spider finishes, the Alerts pane will already be populated with passive findings such as missing security headers, information disclosure, and DOM-based XSS indicators.

Two alert views are available:

- **Left pane alerts**: Issues found specifically on the current page you are viewing in the browser
- **Right pane alerts**: All alerts found across the entire application

The full alerts list with details is also available under the Alerts tab in the main ZAP UI. Clicking any alert shows its description, risk level, confidence, the affected URL, and remediation guidance.

***

## Active Scanner

Click the Active Scan button on the right pane to launch a full active scan. If no spider has been run yet, ZAP runs one automatically first to build the site tree before proceeding.

The active scanner tests all identified pages and HTTP parameters with a range of attack payloads, looking for vulnerabilities including:

- SQL injection
- Command injection
- XSS (reflected and stored)
- Path traversal
- Server-side includes
- Remote file inclusion

Active scanning takes significantly longer than passive scanning because it sends many crafted requests to each endpoint. Progress is tracked in the HUD and in the main UI, where the Logger view shows every request ZAP generates.

As the scan runs, the Alerts pane continues to populate in real time. High severity alerts are the priority: clicking the High Alerts button filters to only those findings, and clicking an individual alert shows the full detail view including the exact request and response ZAP used to confirm the vulnerability. From there you can replay the request directly through the HUD or the Request Editor for manual verification.

***

## Reporting

Generate a report from `Report > Generate HTML Report` in the top menu bar. ZAP also supports XML and Markdown export formats for teams that feed findings into tracking systems or documentation pipelines.

The generated HTML report organises all findings by severity and includes details on each identified vulnerability. As with Burp's report, this output should be treated as supporting evidence rather than a final deliverable. The raw scan report works well as an appendix but the core penetration test report still requires human analysis, business impact context, and prioritised remediation recommendations that no automated scanner can provide.

***

## ZAP vs Burp Scanner

| Feature | ZAP Scanner | Burp Scanner |
|---------|------------|-------------|
| Cost | Free | Pro only |
| Spider | Standard + Ajax Spider | Crawler only |
| Passive scan | Automatic during spider | Manual trigger or during audit |
| Active scan | Full, no throttle | Full, highly polished |
| Report formats | HTML, XML, Markdown | HTML |
| Login handling | Basic | Record login sequence |
| Update frequency | Community-driven | PortSwigger research team |

ZAP Scanner is a strong choice for engagements where Burp Pro is not available, and its Ajax Spider gives it a genuine edge over Burp's crawler for JavaScript-heavy applications. For mature enterprise web application assessments where the depth and polish of Burp's active scanner and the quality of its advisory content matter, Burp Pro remains the industry preference.
