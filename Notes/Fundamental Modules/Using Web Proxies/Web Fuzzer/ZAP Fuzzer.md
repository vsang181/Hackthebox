# ZAP Fuzzer

ZAP's built-in fuzzer covers most of the same ground as Burp Intruder with one critical advantage: no rate throttling at any tier. This makes it the practical choice for large wordlists when you are not running Burp Pro.

***

## Starting a Fuzz Attack

Capture a request through the proxy, locate it in the History tab, right-click it, and select `Attack > Fuzz`. This opens the Fuzzer window with the full request visible and ready to configure.

***

## Fuzz Locations

Equivalent to Burp's payload positions. Select the word in the request you want to replace with payloads (for directory fuzzing, select the directory name in the path), then click "Add" on the right pane. A green marker appears over the selected text confirming the fuzz point is set, and the Payloads window opens automatically.

***

## Payloads

ZAP offers eight payload types. The most useful for general web testing are:

| Payload Type | Use Case |
|-------------|---------|
| File | Load a custom wordlist from disk |
| File Fuzzers | Use built-in wordlists from ZAP's bundled databases |
| Numberzz | Generate numeric sequences with configurable increments |
| Strings | Provide a manual list of specific values |

The "File Fuzzers" type is a genuine advantage over Burp Community. ZAP ships with built-in wordlists sourced from dirbuster and other databases, so you can start a directory or parameter fuzz immediately without needing SecLists or any external file. Additional wordlist databases can be installed from the ZAP Marketplace. For the directory fuzzing example, selecting the first dirbuster wordlist from File Fuzzers is sufficient.

***

## Processors

Processors apply transformations to each payload item before it is sent. Available options include:

- URL Decode/Encode
- Base64 Decode/Encode
- MD5 / SHA-1 / SHA-256 / SHA-512 Hash
- Prefix String (prepend a custom string to every payload)
- Postfix String (append a custom string to every payload)
- Script (run a custom ZAP script on every payload)

For directory fuzzing, add a URL Encode processor to handle any special characters in wordlist entries cleanly. Use the "Generate Preview" button to verify the final payload looks correct before committing. Prefix and Postfix String are particularly useful when fuzzing file paths where you want to append extensions like `.php` or `.bak` to every word automatically.

***

## Options

Key settings to configure before starting:

- **Concurrent Scanning Threads**: Set to 20 for a reasonably fast scan without overwhelming the server. Increase this if the target can handle more concurrent connections, or reduce it if you are worried about triggering rate limits or IDS alerts.
- **Depth first vs Breadth first**: Controls iteration order when multiple fuzz positions exist.
  - Depth first: Exhausts all payloads on position 1 before moving to position 2. Best for credential stuffing where you want to try all passwords for one user before moving to the next.
  - Breadth first: Tries each payload across all positions before moving to the next payload. Better for spraying a single password across many accounts.

***

## Running and Reading Results

Click "Start Fuzzer". Once running, sort the results table by the Response Code column to surface 200 OK hits immediately. For directory fuzzing, a 200 response confirms the directory exists and is accessible.

Other columns worth monitoring depending on your attack type:

| Column | Useful For |
|--------|-----------|
| Response Code | Directory/file enumeration, authentication testing |
| Size Resp. Body | Detecting different content (e.g., valid login vs. failed login pages that return the same 200 code) |
| RTT (Round Trip Time) | Time-based blind SQL injection, where a successful payload causes a deliberate server delay |

Right-click any interesting result to open the full request and response for detailed inspection, or send it to the Request Editor for further manual testing.

***

## ZAP Fuzzer vs Burp Intruder

| Feature | ZAP Fuzzer | Burp Intruder (Community) | Burp Intruder (Pro) |
|---------|-----------|--------------------------|-------------------|
| Speed | Unlimited | 1 req/sec | Unlimited |
| Built-in wordlists | Yes | No | No |
| Payload processors | Yes (basic) | Yes (advanced) | Yes (advanced) |
| Attack types | Depth/breadth | Sniper/Cluster Bomb/etc. | Sniper/Cluster Bomb/etc. |
| Grep/match rules | Partial (sort by code) | Yes (Grep-Match) | Yes (Grep-Match) |
| Cost | Free | Free | Paid |

For large wordlist fuzzing on a budget, ZAP Fuzzer is the better tool. For more granular control over attack types, payload processing chains, and match rules, Burp Intruder Pro is worth the investment on professional engagements.
