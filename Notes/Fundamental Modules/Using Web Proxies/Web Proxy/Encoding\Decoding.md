# Encoding and Decoding

When crafting and modifying HTTP requests manually, encoding is not optional. Sending unencoded special characters in request data can break the request structure entirely or cause the server to misinterpret the input. Both Burp and ZAP provide built-in encoding tools so you never need to leave the proxy to handle this.

***

## Why URL Encoding Matters

HTTP has reserved characters that carry specific meaning within a request. If these appear unencoded in your payload, the server (or an intermediary) will interpret them structurally rather than as part of your data:

| Character | Unencoded Meaning | Encoded Form |
|-----------|------------------|-------------|
| Space | Ends the request data | `%20` or `+` |
| `&` | Delimits parameters | `%26` |
| `#` | Fragment identifier | `%23` |
| `=` | Separates key from value | `%3D` |

A command injection payload like `;ls;` contains characters that may need encoding depending on the context. Sending it raw works in some cases, but encoding ensures the server receives exactly what you intended.

***

## URL Encoding in Burp

In Burp Repeater, select the text you want to encode, right-click, and choose `Convert Selection > URL > URL-encode key characters`, or press `CTRL+U` as a shortcut. Burp also supports live encoding as you type, which you can enable by right-clicking in the request editor. There are also extended options like Full URL-Encoding and Unicode URL Encoding for payloads with a high density of special characters.

ZAP handles URL encoding automatically in the background before sending requests, so you typically do not need to manually encode in ZAP's Request Editor.

***

## Decoding in Burp: Decoder Tab

Burp's Decoder tab accepts any string and lets you apply encoding or decoding operations sequentially. For example, if you encounter a cookie value like:

```
eyJ1c2VybmFtZSI6Imd1ZXN0IiwgImlzX2FkbWluIjpmYWxzZX0=
```

Paste it into Decoder and select `Decode as > Base64`. The output reveals:

```json
{"username":"guest", "is_admin":false}
```

Decoder also supports chaining: the output pane at the bottom has its own encoding selector, so you can decode base64 and then immediately re-encode the result as something else without copying between fields.

Newer versions of Burp also include the Inspector tool, accessible directly within Proxy and Repeater, which provides quick encoding and decoding inline without needing to switch to the Decoder tab.

***

## Decoding in ZAP: Encoder/Decoder/Hash

Open it with `CTRL+E`. ZAP automatically runs the input through multiple decoders simultaneously and displays all results in the Decode tab, so you do not need to know the encoding type upfront. You can also create custom tabs using the "Add New Tab" button and populate them with whichever encoder/decoder combinations you use most frequently.

***

## Encoding for Attack Purposes

Decoding is only half the workflow. The more impactful skill is encoding modified values back to their original format and injecting them into requests. Using the cookie example above, the attack chain looks like this:

1. Decode the base64 cookie to reveal `{"username":"guest", "is_admin":false}`
2. Modify the values to `{"username":"admin", "is_admin":true}`
3. Re-encode the modified string as base64
4. Replace the original cookie value in your Repeater request with the new encoded string
5. Send and observe whether the application grants elevated privileges

This workflow applies to any encoded data you encounter: JWT payloads, serialised objects, hidden form fields, and API tokens. The encoding format supported by both tools covers the most common types you will encounter:

- HTML encoding
- Unicode encoding
- Base64
- ASCII hex
- URL encoding (standard and full)

Being able to encode and decode directly within the proxy removes the need for external tools or browser developer consoles during an assessment, keeping your workflow contained and efficient.
