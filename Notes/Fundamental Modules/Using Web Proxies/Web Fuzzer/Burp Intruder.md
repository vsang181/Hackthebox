# Burp Intruder

Burp Intruder is the built-in fuzzing and brute-forcing engine within Burp Suite. It automates the process of sending a large number of modified requests to a target, iterating through a wordlist or payload set to discover directories, parameters, credentials, and other assets. The key limitation to be aware of upfront is that the free Community version is throttled to 1 request per second, making it practical only for short, targeted attacks. Burp Pro removes this restriction entirely.

***

## Sending a Request to Intruder

Start by locating your target request in Proxy History, then either right-click and select "Send to Intruder" or press `CTRL+I`. Navigate to the Intruder tab with `CTRL+SHIFT+I`.

***

## Positions Tab

The Positions tab is where you define which part of the request will be replaced with your payload on each iteration. For directory fuzzing, you want to test `GET /DIRECTORY/` where `DIRECTORY` is the fuzz point.

Mark the payload position by either:
- Wrapping the target word with `§` symbols manually
- Selecting the word and clicking the "Add §" button

Intruder supports four attack types that determine how multiple payload positions are handled:

| Attack Type | Behaviour |
|------------|-----------|
| Sniper | One payload list, one position at a time. Best for single-parameter fuzzing |
| Battering Ram | Same payload inserted into all positions simultaneously |
| Pitchfork | Multiple payload lists, one per position, iterated in parallel |
| Cluster Bomb | Multiple payload lists, all combinations tested. Good for credential stuffing |

For directory fuzzing with a single position, Sniper is the right choice.

***

## Payloads Tab

### Payload Type

For most fuzzing tasks, "Simple List" is the starting point. Other useful types include:

- **Runtime File**: Loads the wordlist line by line during the scan rather than all at once. Use this for large wordlists to avoid excessive memory consumption by Burp
- **Character Substitution**: Tests permutations of a character replacement map, useful for password mutation attacks

The full list of payload types is documented at [portswigger.net/burp/documentation/desktop/tools/intruder/configure-attack/payload-types](https://portswigger.net/burp/documentation/desktop/tools/intruder/configure-attack/payload-types).

### Payload Configuration

Click "Load" to import a wordlist file. For directory fuzzing, `/opt/useful/seclists/Discovery/Web-Content/common.txt` is a solid starting point. You can combine multiple wordlists by loading them one after another, and items from each are appended to the same list.

### Payload Processing

Processing rules let you filter or transform items from the wordlist before they are sent. To skip lines that start with `.` (common in wordlists for hidden files/directories that would return 404 anyway), add a rule:

- Rule type: Skip if matches regex
- Pattern: `^\..*$`

This cleans up the results by removing noise before the scan even runs.

### Payload Encoding

Leave URL encoding enabled by default. Disable it only if you are sending payloads that are already encoded or if you are testing how a server handles raw special characters.

***

## Settings Tab

Key options worth configuring before running an attack:

- **Grep - Match**: Flags responses containing a specific string. For directory fuzzing, clear the default list and add `200 OK` to highlight successful hits at a glance. Disable "Exclude HTTP Headers" since the status code lives in the header.
- **Grep - Extract**: Pulls a specific portion of the response body into the results table. Useful when responses are long and you only care about one field.
- **Number of retries / Pause before retry**: Set both to 0 to keep the attack moving without waiting on failed connections.
- **Resource Pool**: Controls network concurrency. Leave at default for standard attacks.

***

## Running the Attack

Click "Start Attack". The results table shows each payload alongside its response status code, length, and any Grep Match flags. Click the `200 OK` column header to sort hits to the top.

For the directory fuzzing example, a hit on `/admin/` with a 200 response confirms the directory exists. You can then right-click that result and send it directly to Repeater for further manual investigation.

***

## Practical Use Cases

Intruder is not limited to directory fuzzing. The same workflow applies to:

- **Credential brute-forcing**: Place payload positions on username and password fields, use a credential wordlist, and look for responses that differ from the failed login response
- **Parameter fuzzing**: Discover hidden or undocumented parameters by fuzzing parameter names and watching for behavioural changes in the response
- **Password spraying against AD-integrated applications**: OWA, SSL VPN portals, RDS, and Citrix all accept HTTP-based authentication that Intruder can target. A single password sprayed across many usernames avoids lockout thresholds that a per-account brute-force would trigger
- **Session token analysis**: Send the same request many times and collect tokens to feed into Burp Sequencer for entropy analysis

The throttling in the Community version makes Intruder impractical for large wordlists, which is where ZAP's fuzzer (covered in the next section) becomes the better choice as it has no rate limits on any tier.
