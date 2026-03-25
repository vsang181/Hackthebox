## Intro to Web Proxies

Web proxies sit between a browser or mobile application and a back-end server, capturing all traffic flowing in both directions. This makes them the single most important tool category in web application penetration testing, since virtually all modern apps rely on constant HTTP/HTTPS communication with back-end servers to function.

Unlike network sniffers such as Wireshark, which capture all local traffic at the packet level, web proxies focus specifically on web ports (primarily HTTP/80 and HTTPS/443) and present requests and responses in a readable, manipulable format. The ability to intercept, modify, and replay individual requests is what makes them indispensable for testing how a back-end server responds to unexpected or malicious input.

***

## What Web Proxies Are Used For

Beyond simple request capture, web proxies support a broad range of testing tasks:

- Web application vulnerability scanning
- Web fuzzing (automated input testing across parameters)
- Web crawling and application mapping
- Web request analysis and modification
- Web configuration testing
- Assisting with code reviews by showing exactly what an application sends and receives

***

## Burp Suite vs ZAP

These are the two dominant tools in the space, and understanding the trade-offs helps you pick the right one for each engagement:

| Feature | Burp Suite | OWASP ZAP |
|---------|-----------|-----------|
| Cost | Free (Community) / Paid (Pro/Enterprise) | Fully free and open-source |
| Active Scanner | Pro only | Free, no throttling |
| Intruder speed | Throttled in Community | No throttle |
| Built-in Browser | Built-in Chromium | External browser required |
| Extensions | Some require Pro | All free via marketplace |
| Best for | Corporate pentests, polished UI | Budget-conscious, CI/CD, open environments |

***

## Burp Suite

Burp is the industry standard and is what most web pentesters reach for first. The Community edition covers the core workflow for most assessments, including intercepting and replaying requests via Proxy and Repeater, manual parameter manipulation, and basic spidering.

The paid Pro tier removes Intruder rate throttling, unlocks the active scanner, and enables certain BApp Store extensions. PortSwigger offers a free Pro trial for educational or business email addresses, which is worth activating if you want to practice with the full toolset during training.

***

## OWASP ZAP

ZAP is entirely community-maintained with no paywalled features. Its key advantages over Burp Community are:

- No scan throttling or rate limits on any feature
- Full active scanner available at no cost
- Built-in REST API and daemon mode for CI/CD pipeline integration
- WebSocket interception support natively
- A growing set of community-contributed extensions that replicate many Burp Pro capabilities

***

## Setting Up the Testing Workflow

The basic setup for both tools follows the same pattern:

1. Configure the proxy listener (default `127.0.0.1:8080` for both tools)
2. Point your browser's proxy settings at that address and port
3. Install the proxy's CA certificate in your browser to intercept HTTPS traffic without SSL errors
4. Enable intercept mode to pause and inspect individual requests before forwarding them

Once this is in place, every HTTP and HTTPS request your browser makes flows through the proxy, giving you full visibility and control over all application traffic. The practical approach during training is to learn both tools. They share similar concepts and workflows, so knowledge of one transfers readily to the other. Burp tends to be preferred for manual, in-depth corporate engagements, while ZAP is the natural choice when budget is a constraint or when automation and open-source extensibility are priorities.
