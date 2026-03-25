# Burp Scanner

Burp Scanner is a Pro-only feature that combines a web crawler, passive analysis engine, and active vulnerability scanner into a single workflow. It is one of the most capable automated web scanners available and is regularly updated with new vulnerability checks by the PortSwigger research team.

***

## Defining Scope

Before scanning anything, define your target scope so Burp focuses its resources on the right assets and ignores everything else.

Navigate to `Target > Site Map` to see all URLs Burp has observed through the proxy. Right-click any item and select "Add to Scope" to include it. When you add your first item, Burp will offer to restrict all features to in-scope items only, which prevents accidental scanning of out-of-scope assets.

To exclude dangerous endpoints like logout functions that would terminate your session mid-scan, right-click them and select "Remove from Scope". The full scope configuration, including regex-based include/exclude patterns, lives at `Target > Scope`.

***

## Crawler

The crawler maps the target by following every link and form it finds, building a comprehensive site map without you needing to browse manually. It does not perform fuzzing for undiscovered paths; it only follows links that exist in the pages it visits. For discovering unlisted directories, Burp Intruder or Content Discovery handles that separately.

To start a crawl, click "New Scan" on the Dashboard tab. The scope you defined will be pre-populated. Select "Crawl" (without audit) for a mapping-only run.

In the Scan Configuration tab, use "Select from library" to choose a preset rather than building from scratch. "Crawl strategy - fastest" is a reasonable starting point for most assessments.

The Application Login tab allows you to supply credentials or record a manual login sequence so the crawler can access authenticated sections of the application. An unauthenticated crawl will miss anything behind a login wall.

Crawl progress appears in the Dashboard under Tasks. Once complete, the updated site map reflects everything the crawler discovered.

***

## Passive Scanner

A passive scan analyses pages already visited without sending any new requests. It inspects source code and response content to identify potential issues like missing security headers, DOM-based XSS indicators, and information disclosure. Because it does not actively probe the application, it cannot confirm vulnerabilities, only flag candidates.

To run one, right-click a target in Site Map or a request in Proxy History and select "Do passive scan" or "Passively scan this target". Results appear in the Issue Activity pane on the Dashboard with severity and confidence ratings.

When reviewing results, prioritise:

- **High severity + Certain/Firm confidence**: Investigate immediately
- **Medium severity + Firm confidence**: Worth manual verification
- **Low/Informational**: Document but deprioritise unless the application is highly sensitive

***

## Active Scanner

The active scanner is the most thorough option and runs the following sequence:

1. Crawl the target and fuzz for undiscovered pages
2. Run a passive scan across all discovered pages
3. Send verification requests for all passive scan candidates
4. Perform JavaScript analysis for client-side vulnerabilities
5. Fuzz identified insertion points and parameters for XSS, SQLi, command injection, and other common vulnerabilities

Start an active scan from the Dashboard with "New Scan", selecting "Crawl and Audit". The Audit Configuration tab lets you narrow the vulnerability classes checked. For targeted assessments focused on critical exploitability, "Audit checks - critical issues only" reduces noise and scan time significantly.

Active scan progress and all generated requests are visible in the Logger tab, which shows every request Burp made during the scan.

***

## Reading Results

Once scanning completes, the Issue Activity pane on the Dashboard displays all findings. Use the filter controls to narrow by severity and confidence. A finding like OS command injection ranked High severity and Firm confidence is a priority item.

Clicking any finding shows:

- A written advisory explaining the vulnerability
- The exact request that triggered it
- The server response that confirmed it
- Remediation guidance

***

## Reporting

When the assessment is complete, go to `Target > Site Map`, right-click the target, and select `Issues > Report issues for this host`. Burp generates an HTML report that can be filtered by severity and confidence, includes proof-of-concept request/response pairs, and provides remediation recommendations for each finding.

One important professional note: Burp's exported report should never be submitted directly to a client as the final deliverable. It is supplementary data. The value of a professional penetration test report lies in the context, narrative, business impact analysis, and remediation prioritisation that a skilled tester adds around the raw scan data. The Burp report works well as an appendix for clients who need raw findings for their tracking dashboards or remediation teams, but the main report requires human analysis and writing.
