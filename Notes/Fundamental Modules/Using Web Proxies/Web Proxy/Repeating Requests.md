# Repeating Requests

Intercepting and forwarding a request manually every time you want to test a different payload is workable for a single test but completely impractical for anything iterative. Request repeating solves this by letting you resend any historical request with modifications directly from within the proxy, bypassing the intercept-forward cycle entirely.

***

## Proxy History

Before repeating anything, you need to locate the request you want to work with. Both tools maintain a full log of every HTTP request that has passed through the proxy.

- **Burp**: `Proxy > HTTP History`
- **ZAP**: The History tab at the bottom of the main UI, or the bottom History pane in the HUD

Both tools support filtering and sorting on the history, which becomes essential once a target application generates hundreds of requests during normal browsing. Filters let you narrow down by URL, method, status code, or response size to find the specific request you are after.

One useful difference between the two: Burp preserves both the original and the edited version of any modified request. If a request was changed before forwarding, the pane header will read "Original Request" and you can switch to "Edited Request" to see exactly what was sent. ZAP only shows the final modified version.

Both tools also maintain a separate WebSockets history for asynchronous connections made after the page loads, which is relevant for more advanced assessments but outside the scope of this module.

***

## Repeating in Burp: Repeater

Once you locate the target request in HTTP History:

1. Press `CTRL+R` to send it to the Repeater tab
2. Navigate to Repeater with `CTRL+SHIFT+R`
3. Modify the request body, headers, or any parameter directly in the editor
4. Click Send and the response appears in the right pane immediately

There is no page reload, no browser interaction, and no intercept toggle needed. You can chain multiple commands rapidly by editing the payload value and hitting Send each time, getting output back instantly in the same window.

A useful shortcut: right-click any request and select "Change Request Method" to toggle between POST and GET without rewriting the request from scratch.

***

## Repeating in ZAP: Request Editor

In ZAP, right-click the target request in the History tab and select "Open/Resend with Request Editor". This opens a dedicated editor window where you can:

- Modify any part of the request directly
- Use the Method dropdown to switch HTTP methods without manual editing
- Click Send to fire the request and see the response in the same window

The Request Editor window layout can be adjusted using the display buttons in the top-right corner of the window. Switching to a side-by-side view for request and response (rather than separate tabs) tends to be more efficient when iterating through payloads.

### ZAP HUD Replay

Within the pre-configured ZAP browser, locate the request in the HUD's bottom History pane and click it to open the Request Editor. You then have two replay options:

- **Replay in Console**: Response is returned in the HUD overlay within the browser window
- **Replay in Browser**: Response is rendered as a full page in the browser, useful when you want to see how the output actually looks

***

## Practical Comparison

| Feature | Burp Repeater | ZAP Request Editor | ZAP HUD Replay |
|---------|--------------|-------------------|----------------|
| Access shortcut | `CTRL+R` from history | Right-click > Open/Resend | Click request in HUD history |
| Method switching | Right-click > Change Method | Method dropdown | Method dropdown |
| Response display | Inline right pane | Inline or tabbed | Console or full browser |
| Original vs edited diff | Yes | No | No |

***

## URL Encoding Note

Looking at the POST request body from the command injection example, you will notice the data is URL-encoded (spaces become `+` or `%20`, special characters become their percent-encoded equivalents). When modifying payloads in Repeater or the Request Editor, you need to be aware of whether the application expects encoded or raw input. Sending an unencoded payload when the server expects encoded data (or vice versa) will often cause the request to fail or produce unexpected results. This is covered in more depth in the next section on encoding.
