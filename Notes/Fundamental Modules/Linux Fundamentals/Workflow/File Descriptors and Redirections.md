# File Descriptors and Redirections

To work efficiently in the Linux shell, you need to understand how **input and output are handled internally**. This is where file descriptors come in. They are fundamental to how data moves between commands, files, and the operating system, and mastering them will significantly improve how you structure commands and automate tasks 

A **file descriptor (FD)** is a reference maintained by the kernel that represents an open input or output resource. This could be a file, a socket, or even a running process. On Windows systems, you may hear a similar concept referred to as a *file handle*.

A useful mental model is a coatroom ticket. When you hand in your coat, you receive a ticket. That ticket does not *contain* the coat, but it uniquely identifies where the coat is stored. Whenever you want your coat back, you present the ticket. In the same way, a file descriptor is the operating system’s “ticket” that points to a specific I/O resource.

Understanding file descriptors becomes essential once you start redirecting output, suppressing errors, or chaining commands together.

---

## Standard File Descriptors

By default, every Linux process starts with three file descriptors:

| File Descriptor | Name   | Description     |
| --------------- | ------ | --------------- |
| `0`             | STDIN  | Standard input  |
| `1`             | STDOUT | Standard output |
| `2`             | STDERR | Standard error  |

These streams exist automatically and are used constantly, even when you do not explicitly reference them.

---

## STDIN and STDOUT

Consider the `cat` command with no arguments. In this case, it reads from **STDIN** and writes to **STDOUT**.

When you type input and press `ENTER`, that input is sent to `cat` via STDIN (FD 0). `cat` then immediately writes it back to STDOUT (FD 1), which is displayed in the terminal.

This simple behaviour demonstrates how data flows through standard streams.

---

## STDOUT and STDERR

Now consider a command that produces both valid output and errors. For example:

```text
find /etc/ -name shadow
```

This command may return:

* Valid results (STDOUT)
* Permission errors (STDERR)

Both streams are displayed in the terminal by default, but they are **separate channels**.

---

## Redirecting STDERR

You can redirect error output (FD 2) to the *null device*, which discards everything sent to it:

```text
find /etc/ -name shadow 2>/dev/null
```

Now only valid output is shown. All permission errors are silently discarded.

This is extremely useful during enumeration, where permission errors are expected and add noise.

---

## Redirecting STDOUT to a File

You can redirect standard output (FD 1) to a file using `>`:

```text
find /etc/ -name shadow 2>/dev/null > results.txt
```

This command:

* Suppresses errors
* Writes only valid results into `results.txt`

If the file does not exist, it is created. If it already exists, it is **overwritten without warning**.

---

## Redirecting STDOUT and STDERR Separately

You can also redirect both streams independently:

```text
find /etc/ -name shadow 2> stderr.txt 1> stdout.txt
```

This gives you:

* Clean results in `stdout.txt`
* Error messages in `stderr.txt`

Separating output like this is particularly useful for documentation and reporting.

---

## Redirecting STDIN

Redirection also works in the opposite direction. The `<` operator feeds a file into STDIN:

```text
cat < stdout.txt
```

Here, the contents of `stdout.txt` are passed to `cat` as if you typed them manually.

---

## Appending Instead of Overwriting

To append output to an existing file instead of overwriting it, use `>>`:

```text
find /etc/ -name passwd >> stdout.txt 2>/dev/null
```

This adds new results to the end of `stdout.txt`.

---

## Redirecting Input Streams with `<<` (Here Documents)

The double lower-than operator (`<<`) allows you to provide multi-line input directly in the terminal. This is known as a **here document**.

Example:

```text
cat << EOF > stream.txt
Hack The Box
EOF
```

Everything between `<< EOF` and `EOF` is treated as STDIN and written into `stream.txt`.

---

## Pipes

Pipes (`|`) connect the STDOUT of one command directly to the STDIN of another.

Example:

```text
find /etc/ -name "*.conf" 2>/dev/null | grep systemd
```

Here:

* `find` produces output
* `grep` filters that output
* Errors are discarded

Pipes allow you to build powerful command chains without temporary files.

You can continue chaining commands:

```text
find /etc/ -name "*.conf" 2>/dev/null | grep systemd | wc -l
```

This counts how many matching results were found.

---

## Why This Matters

Once you understand file descriptors, redirection, and pipes, the shell stops feeling like a collection of isolated commands and starts behaving like a **data-processing pipeline**.

This allows you to:

* Suppress noise
* Extract only relevant information
* Automate complex tasks
* Write cleaner, more reliable command chains

These concepts appear constantly throughout penetration testing, from enumeration to reporting. The time you spend mastering them now will save you significant effort later and give you much finer control over how you interact with Linux systems.
