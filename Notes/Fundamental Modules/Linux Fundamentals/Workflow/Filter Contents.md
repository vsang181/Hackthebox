# Filtering and Viewing File Contents

In the previous section, you learned how to redirect output between commands. Now you will build on that knowledge by **reading and filtering files directly from the command line**, without opening a text editor. This is a core skill when dealing with logs, configuration files, and large outputs during enumeration and analysis 

Linux provides several lightweight tools that allow you to inspect, filter, and transform text efficiently. You will use these constantly, often chaining them together to extract exactly the information you need.

---

## Pagers: `more` and `less`

Pagers allow you to view file contents **one screen at a time**. This is essential when working with large files that do not fit on a single terminal screen.

### Using `more`

You can pipe output into `more` to begin reading from the top of the file:

```text
cat /etc/passwd | more
```

The `/etc/passwd` file acts like a directory of system users. It contains:

* Username
* User ID (UID)
* Group ID (GID)
* Home directory
* Default shell

When `more` opens, you start at the top of the file. The `--More--` indicator shows that additional content exists.

* Press `SPACE` to advance
* Press `q` to quit

When you exit `more`, the output **remains visible** in the terminal.

---

### Using `less`

The `less` pager provides the same basic functionality as `more`, but with additional features such as backward navigation and searching.

```text
less /etc/passwd
```

The display initially looks similar, but there is an important difference:

* When you quit `less` using `q`, the output **does not remain** on the screen

This makes `less` preferable when you want a clean terminal after viewing a file.

In practice, `less` is almost always the better choice.

---

## Viewing Specific Sections of a File

Sometimes you only care about the beginning or the end of a file.

### `head`

By default, `head` prints the first ten lines:

```text
head /etc/passwd
```

This is useful for quickly inspecting headers or initial entries.

### `tail`

The `tail` command prints the last ten lines:

```text
tail /etc/passwd
```

This is especially useful for log files, where new entries are usually appended at the end.

---

## Sorting Output with `sort`

Command output is rarely sorted in a helpful way by default. To organise results alphabetically or numerically, you can use `sort`.

```text
cat /etc/passwd | sort
```

After sorting, entries no longer start with `root`, but instead appear in alphabetical order. This often makes patterns and anomalies easier to spot.

---

## Searching with `grep`

One of the most powerful filtering tools is `grep`. It allows you to search for lines that match specific patterns.

### Example: Users with `/bin/bash`

```text
cat /etc/passwd | grep "/bin/bash"
```

This filters the file to show only users whose default shell is `/bin/bash`.

### Excluding Matches with `-v`

To exclude specific patterns, use the `-v` option:

```text
cat /etc/passwd | grep -v "false\|nologin"
```

This removes users whose shells are disabled.

---

## Extracting Fields with `cut`

Many system files use delimiters. In `/etc/passwd`, fields are separated by colons (`:`).

To extract specific fields, use `cut`:

```text
cat /etc/passwd | grep -v "false\|nologin" | cut -d":" -f1
```

Here:

* `-d ":"` sets the delimiter
* `-f1` selects the first field (username)

---

## Replacing Characters with `tr`

The `tr` command replaces characters in a stream.

Example: replace colons with spaces:

```text
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " "
```

This makes the output easier to read or process further.

---

## Formatting Output with `column`

To align output into neat columns, use `column -t`:

```text
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | column -t
```

This produces a structured, table-like output that is far easier to interpret visually.

---

## Advanced Field Processing with `awk`

Sometimes fields are inconsistent, making `cut` unreliable. This is where `awk` becomes useful.

Example: print the first field and the last field only:

```text
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}'
```

Here:

* `$1` is the first column
* `$NF` is the last column

This allows you to handle irregular spacing cleanly.

---

## Editing Streams with `sed`

The `sed` tool is a stream editor used to modify text on the fly.

Example: replace the word `bin` with `HTB` globally:

```text
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}' | sed 's/bin/HTB/g'
```

Breaking it down:

* `s` → substitute
* `bin` → pattern to replace
* `HTB` → replacement
* `g` → replace all occurrences

---

## Counting Results with `wc`

To count the number of matching lines, use `wc -l`:

```text
cat /etc/passwd | grep -v "false\|nologin" | tr ":" " " | awk '{print $1, $NF}' | wc -l
```

This returns the total number of lines produced by the filtered output.
