# Filtering Results in Ffuf

Ffuf's default behaviour filters out only HTTP 404 responses, which is not enough when you are vhost fuzzing and every request returns 200. The solution is to identify the baseline "noise" response and actively exclude it, leaving only genuine findings.

***

## Matchers vs Filters

Ffuf gives you two opposing ways to control output:

- Matchers (`-m` flags) work as an allowlist: only show results that match the condition
- Filters (`-f` flags) work as a blocklist: hide results that match the condition

For vhost fuzzing, filters are the right tool because you know what the false positive looks like (size 900) but you do not know what a real vhost response will look like. You cannot match on an unknown value, but you can filter out the known noise.

Full reference:

| Flag | Type | Description |
|------|------|-------------|
| `-mc` | Matcher | Status codes to keep (default: 200,204,301,302,307,401,403) |
| `-ml` | Matcher | Match exact line count |
| `-ms` | Matcher | Match exact response size |
| `-mw` | Matcher | Match exact word count |
| `-mr` | Matcher | Match regex pattern in response body |
| `-fc` | Filter | Exclude by status code |
| `-fl` | Filter | Exclude by line count |
| `-fs` | Filter | Exclude by response size |
| `-fw` | Filter | Exclude by word count |
| `-fr` | Filter | Exclude by regex pattern in response body |

***

## Applying the Filter

Once you have identified the baseline response size from a test run (900 bytes in this case), add `-fs 900` to strip it from results:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://academy.htb:PORT/ \
     -H 'Host: FUZZ.academy.htb' \
     -fs 900
```

The single result that survives is `admin.academy.htb`, confirming it serves different content from the default vhost. Its size of 0 means the page body is empty, but the fact that it responds differently from everything else is what matters.

***

## Confirming the Discovery

After finding the vhost, two steps are needed before you can interact with it:

```bash
# 1. Add it to /etc/hosts
sudo sh -c 'echo "SERVER_IP  admin.academy.htb" >> /etc/hosts'

# 2. Visit it in your browser or with curl
curl http://admin.academy.htb:PORT/
```

The confirmation tests used here are worth understanding as a method:

- Visiting `admin.academy.htb` returns an empty page rather than the `academy.htb` homepage, confirming a different vhost is being served
- Visiting `/blog/index.php` on `admin.academy.htb` returns a 404, confirming the two vhosts have entirely separate directory structures and content

***

## Determining the Baseline Before Filtering

When you are not sure what the noise response size is, run the scan once without any filter first and observe the results. If every result shares identical size, word count, or line count values, that is your baseline. You can filter on any of those dimensions. Size tends to be the most reliable because it is specific, while word count can occasionally vary slightly for the same page due to whitespace differences.

For cases where the false positive varies in size (uncommon but possible with dynamic error pages), `-fw` or `-fl` can be more stable. If the default error page contains a specific string like "Not Found" or "Default Page", `-fr` with a regex match is the cleanest approach.

***

## Next Step: Recursive Scan on Admin

Since the admin vhost is confirmed, the natural next move is running a recursive page fuzzing scan against it:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://admin.academy.htb:PORT/FUZZ \
     -recursion \
     -recursion-depth 1 \
     -e .php \
     -v \
     -fs 0
```

The `-fs 0` here filters out the empty index pages (size 0) that we already know about, so only pages with actual content surface in the results. Each finding on the admin vhost is likely more sensitive than anything found on the main domain, since admin panels are the typical targets for this kind of enumeration.
