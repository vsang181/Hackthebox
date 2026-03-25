# Automatic Modification

Manual interception works fine for one-off changes, but when you need the same modification applied to every request or response throughout a session, doing it by hand quickly becomes tedious. Both Burp and ZAP let you define rules that apply transformations automatically without any manual intervention.

***

## Automatic Request Modification

### Burp Match and Replace

Navigate to `Proxy > Proxy Settings > HTTP Match and Replace Rules` and click Add. For the User-Agent replacement example, use these settings:

| Field | Value |
|-------|-------|
| Type | Request header |
| Match | `^User-Agent.*$` |
| Replace | `User-Agent: HackTheBox Agent 1.0` |
| Regex match | True |

The regex pattern `^User-Agent.*$` matches the entire User-Agent line regardless of what its current value is, which is why regex is needed here. Once saved and enabled, every outgoing request will have its User-Agent automatically replaced before it leaves the proxy. You can verify this by visiting any page and checking the request in Burp's HTTP history.

### ZAP Replacer

Access ZAP's equivalent via `CTRL+R` or through the Options menu. Click Add and configure:

| Field | Value |
|-------|-------|
| Description | HTB User-Agent |
| Match Type | Request Header (will add if not present) |
| Match String | User-Agent |
| Replacement String | HackTheBox Agent 1.0 |
| Enable | True |

ZAP also supports a "Request Header String" match type that accepts regex patterns, which behaves identically to the Burp approach. The Initiators tab controls where the rule applies. Leaving it on "Apply to all HTTP(S) messages" ensures it runs everywhere.

***

## Automatic Response Modification

The same rule system applies to responses. In the previous section, manually editing `type="number"` in an intercepted response was a one-time fix that disappeared on the next page refresh. Automating it means the field stays editable throughout your entire session.

### Burp

Go back to `Proxy > Proxy Settings > HTTP Match and Replace Rules` and add two new rules:

**Rule 1 - Remove numeric restriction:**

| Field | Value |
|-------|-------|
| Type | Response body |
| Match | `type="number"` |
| Replace | `type="text"` |
| Regex match | False |

**Rule 2 - Extend input length:**

| Field | Value |
|-------|-------|
| Type | Response body |
| Match | `maxlength="3"` |
| Replace | `maxlength="100"` |
| Regex match | False |

After saving both rules, do a forced refresh with `CTRL+SHIFT+R`. The input field will now accept any characters and retain that behaviour on every subsequent refresh without any manual interception required.

### ZAP Replacer Equivalent

Add the same two rules in ZAP's Replacer using these settings:

**Rule 1:**
- Match Type: Response Body String
- Match Regex: False
- Match String: `type="number"`
- Replacement String: `type="text"`

**Rule 2:**
- Match Type: Response Body String
- Match Regex: False
- Match String: `maxlength="3"`
- Replacement String: `maxlength="100"`
