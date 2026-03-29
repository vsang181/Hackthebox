# SQLMap Output Explained

Understanding SQLMap's log messages is as important as running the tool itself. Each line tells you exactly what the tool is doing and why, which helps both with interpreting results and with reporting findings accurately.

***

## Message Reference

### Stability and Parameter Checks

**"target URL content is stable"**
The baseline response does not change between identical requests. This is the foundation for all comparison-based detection: SQLMap needs a consistent baseline to measure deviations caused by injected payloads. Unstable targets (pages with random content, timestamps, or counters) generate noise that SQLMap attempts to filter automatically.

**"GET parameter 'id' appears to be dynamic"**
Changing the parameter value changes the response, indicating the value is processed by backend logic (almost certainly a database query). A static parameter that returns identical responses regardless of value is not linked to any query and is therefore not injectable in the current context.

***

### Detection Phase Messages

**"heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')"**
SQLMap sent an intentionally malformed value (e.g., `?id=1",).))`) and received a MySQL error in response. This is not confirmation of injection, only a strong indicator that warrants further testing. The DBMS fingerprint here allows SQLMap to skip payloads for every other database engine in subsequent tests.

**"heuristic (XSS) test shows that GET parameter 'id' might be vulnerable to XSS attacks"**
A bonus detection running alongside the SQLi heuristic. SQLMap is not an XSS scanner, but it tests for it opportunistically during the same pass. Useful in large-scale assessments where every fast finding matters.

**"reflective value(s) found and filtering out"**
Part of the injected payload is being echoed back in the page response. This is noise that would corrupt comparisons between TRUE and FALSE responses. SQLMap detects and filters reflected values before doing any comparison logic, preventing false positives caused by the reflection itself.

***

### Injection Confirmation Messages

**"GET parameter 'id' appears to be 'AND boolean-based blind' injectable (with --string="luther")"**
Two important details here. First, "appears to be" means it is not yet fully confirmed: SQLMap runs additional logic checks at the end of the scan to eliminate false positives. Second, `--string="luther"` means SQLMap found a constant string in the TRUE response that it can use as a reliable signal, making the boolean detection simpler and more accurate than fuzzy response comparison.

**"time-based comparison requires a larger statistical model, please wait"**
Before time-based blind testing begins, SQLMap collects multiple baseline response times to build a statistical model of normal latency. This model allows it to reliably distinguish a deliberate `SLEEP(5)` delay from natural network jitter, even on high-latency connections. This is why time-based detection is slower than other techniques.

**"automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found"**
UNION detection normally caps at 10 requests per parameter to save time on non-injectable targets. Once another technique confirms the parameter is injectable, that cap is removed and SQLMap runs the full UNION detection range, since success is now likely.

**"'ORDER BY' technique appears to be usable"**
SQLMap confirmed it can use the ORDER BY column-count method before sending full UNION payloads. This allows binary-search style column count detection (e.g., try column 10, then 5, then 3) rather than linear counting from 1 upward, significantly reducing requests needed to identify the correct column count.

***

### Final Output Messages

**"GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)?"**
The most important single message in any SQLMap run. When `--batch` is used, this defaults to N (stop testing other parameters). In a full application assessment where complete coverage is required, answering Y continues testing every other discovered parameter, which may identify additional injection points in the same application.

**"sqlmap identified the following injection point(s) with a total of 46 HTTP(s) requests"**
The definitive confirmation. SQLMap only lists injection points it can provably exploit, not theoretical possibilities. The payload block that follows contains everything needed to manually reproduce the injection, which is exactly what you need for a penetration test report.

**"fetched data logged to text files under '/home/user/.sqlmap/output/www.example.com'"**
All session data, logs, and extracted content persist in this directory. Subsequent runs against the same target read from these session files, allowing SQLMap to skip re-detection and jump straight to extraction. This session persistence is why re-running the same command after an interrupted extraction is fast: the tool already knows the injection type, column count, and DBMS version from the previous session.

***

## Quick Reference: Message Implications for Reporting

| Message | Report Implication |
|---------|------------------|
| Heuristic injectable | Possible finding, pending confirmation |
| Appears to be injectable | Likely finding, false positive checks pending |
| Is vulnerable | Confirmed, reportable injection point |
| Injection points identified | Full proof with payloads for report evidence |
| Data logged to files | Raw extraction available as supporting evidence |

The distinction between "appears to be injectable" and "is vulnerable" matters in reporting. Only the final confirmed findings in the injection points summary block represent provably exploitable vulnerabilities suitable for a report. Heuristic and intermediate messages represent the detection process, not the conclusion.
