# Finding Files and Directories

Being able to locate files and directories efficiently is a critical skill once you gain access to a Linux system. During an assessment, you will often need to identify configuration files, user-created scripts, binaries, or artefacts left behind by administrators. Manually browsing every directory is slow and impractical, especially on large systems. Fortunately, Linux provides several tools that make searching fast and precise 

This section focuses on three essential tools you will use repeatedly: `which`, `find`, and `locate`.

---

## Identifying Available Programs with `which`

The `which` command tells you **where a command is located on the filesystem**, or whether it exists at all. This is extremely useful when checking if common tools such as `curl`, `wget`, `nc`, `python`, or `gcc` are available on the target.

### Syntax

```text
which <command>
```

### Example

```text
which python
```

Example output:

```text
/usr/bin/python
```

If no output is returned, the command is not available in the current `PATH`. This immediately informs your next steps, such as whether you need to upload a binary or rely on alternative tooling.

---

## Searching the Filesystem with `find`

The `find` command is one of the most powerful tools available in Linux. It allows you to search for files and directories and apply detailed filters based on attributes such as name, size, owner, type, and modification time.

### Basic Syntax

```text
find <location> <options>
```

### Advanced Example

```text
find / -type f -name "*.conf" -user root -size +20k -newermt 2020-03-03 -exec ls -al {} \; 2>/dev/null
```

This command searches the entire filesystem and lists configuration files that meet specific criteria.

### Breaking Down the Options

| Option                | Purpose                              |
| --------------------- | ------------------------------------ |
| `-type f`             | Restricts results to regular files   |
| `-name "*.conf"`      | Matches files ending with `.conf`    |
| `-user root`          | Filters files owned by the root user |
| `-size +20k`          | Shows only files larger than 20 KiB  |
| `-newermt 2020-03-03` | Files modified after the given date  |
| `-exec ls -al {} \;`  | Executes a command on each result    |
| `2>/dev/null`         | Suppresses error messages            |

The `-exec` option is particularly important. It allows you to run any command on each matched file. The `{}` placeholder represents the current result, and the escaped semicolon (`\;`) signals the end of the command.

The error redirection (`2>/dev/null`) is not part of `find` itself. It redirects standard error output so that permission errors do not clutter your results.

---

## Faster Searches with `locate`

Searching the entire filesystem with `find` can be slow, especially on large systems. The `locate` command offers a much faster alternative by searching a **local file index database** instead of scanning directories in real time.

### Updating the Database

Before using `locate`, the database should be updated:

```text
sudo updatedb
```

### Searching with `locate`

```text
locate "*.conf"
```

This returns results almost instantly:

```text
/etc/GeoIP.conf
/etc/NetworkManager/NetworkManager.conf
/etc/UPower/UPower.conf
/etc/adduser.conf
<SNIP>
```

### Limitations of `locate`

While `locate` is fast, it has important limitations:

* Results may be outdated if the database is not updated
* Fewer filtering options compared to `find`
* Cannot filter by ownership, size, or date as precisely

Because of this, you should always decide which tool fits your goal:

* Use **`locate`** when you need quick results and approximate matches
* Use **`find`** when you need accuracy, filtering, and control

---

## Key Takeaway

File discovery is not about memorising commands. It is about choosing the **right tool for the situation**.

During real engagements, you will constantly switch between speed and precision. Mastering `which`, `find`, and `locate` gives you the flexibility to adapt quickly, uncover sensitive files, and build a clear picture of the system you are working on.
