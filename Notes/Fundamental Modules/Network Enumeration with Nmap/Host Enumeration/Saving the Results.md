## Saving the Results

When running multiple scans, you should always save results for comparison, documentation, and reporting. Nmap supports three main output formats and a combined option that saves all of them at once.

---

## Output Formats

Nmap can save results in these formats:

* **Normal output (`-oN`)** → `.nmap`
* **Grepable output (`-oG`)** → `.gnmap`
* **XML output (`-oX`)** → `.xml`

You can also use:

* **All formats (`-oA`)** → creates `.nmap`, `.gnmap`, and `.xml` in one run

---

## Saving All Formats at Once (`-oA`)

Example:

```
sudo nmap 10.129.2.28 -p- -oA target
```

| Option        | Description                                                |
| ------------- | ---------------------------------------------------------- |
| `10.129.2.28` | Target host                                                |
| `-p-`         | Scan all TCP ports (1–65535)                               |
| `-oA target`  | Save output as `target.nmap`, `target.gnmap`, `target.xml` |

If you do not provide a full path, Nmap writes the files into your **current working directory**.

Example output files created:

* `target.nmap`
* `target.gnmap`
* `target.xml`

---

## Normal Output (`-oN`)

Human-readable output similar to what you see in the terminal, but saved to a file.

Example:

```
sudo nmap 10.129.2.28 -p- -oN target.nmap
```

Typical use cases:

* Quick reading
* Reporting notes
* Comparing scans manually

---

## Grepable Output (`-oG`)

A more compact, line-oriented format designed to be parsed with tools like `grep`, `awk`, and `cut`.

Example:

```
sudo nmap 10.129.2.28 -p- -oG target.gnmap
```

Typical use cases:

* Fast parsing in bash pipelines
* Extracting open ports quickly
* Feeding results into other scripts/tools

Technical note: `-oG` is still widely seen in older workflows, but many people now prefer parsing XML (`-oX`) because it is more structured and consistent.

---

## XML Output (`-oX`)

A structured format intended for automation, tool ingestion, and report generation.

Example:

```
sudo nmap 10.129.2.28 -p- -oX target.xml
```

Typical use cases:

* Import into reporting tools
* Parsing reliably (no “screen formatting” ambiguity)
* Converting into other formats (HTML, JSON via external tooling, etc.)

---

## Converting XML to HTML (Reporting)

XML output can be transformed into a clean HTML report using `xsltproc`:

```
xsltproc target.xml -o target.html
```

This produces an easy-to-read report that is useful for documentation and for audiences that do not want to interpret raw scan output.

---

## Additional Technical Notes

### Naming conventions that scale

If you are scanning many hosts, use a consistent naming pattern so results do not get overwritten, for example:

* `corp-10.10.10.28-fulltcp`
* `corp-10.10.10.28-top1000`
* `corp-subnet-10.10.10.0_24-discovery`

### Saving “scan context” with the output

Include scan flags that materially change interpretation in the filename (for example `-sU`, `-sV`, `-A`, `-Pn`, `--top-ports`, `-p-`). This avoids confusion later when comparing two files.

### XML is the best “source of truth” for automation

If you plan to parse results programmatically or import into another tool, XML (`-oX`) is usually the most reliable because it preserves structure (host objects, ports, service elements, reasons, timing, etc.).

More details on Nmap output formats:

[https://nmap.org/book/output.html](https://nmap.org/book/output.html)
