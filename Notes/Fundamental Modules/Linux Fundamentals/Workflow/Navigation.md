# Navigation

Navigation is a core skill when working in Linux, much like using a mouse in a graphical Windows environment. You rely on it to move through the filesystem, inspect directories, and work with files efficiently. To do this, you will use a small set of commands repeatedly, often combined with options to tailor the output to exactly what you need 

The best way to learn navigation is by experimenting. In this section, you focus on moving through the filesystem, listing directory contents, and understanding what the shell shows you. You should practise these commands on a local virtual machine. Before experimenting, it is strongly recommended that you take a snapshot so you can easily recover if something breaks.

## Where Am I?

Before moving anywhere, you should always know your current location in the filesystem. You can find this out using the `pwd` command.

```text
pwd
```

Example output:

```text
/home/willy_wonka
```

This tells you the absolute path of your current working directory.

## Listing Directory Contents

To view the contents of a directory, you use the `ls` command.

```text
ls
```

This lists files and directories in the current location:

```text
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
```

By default, this output is minimal. To see more detailed information, you can use the `-l` option.

```text
ls -l
```

This displays additional metadata:

* File type and permissions
* Number of hard links
* Owner and group
* Size
* Last modification date
* Name

Example:

```text
drwxr-xr-x 2 willy_wonka willy_wonka 4096 Jan  2 21:15 Desktop
```

### Understanding the Columns

A long listing (`ls -l`) is structured as follows:

| Column        | Meaning                           |
| ------------- | --------------------------------- |
| `drwxr-xr-x`  | File type and permissions         |
| `2`           | Number of hard links              |
| `willy_wonka` | File owner                        |
| `willy_wonka` | Group owner                       |
| `4096`        | Size (or directory metadata size) |
| `Jan 2 21:15` | Date and time                     |
| `Desktop`     | File or directory name            |

## Hidden Files

Not all files are visible by default. Files starting with a dot (`.`) are hidden. Examples include `.bashrc` and `.bash_history`.

To list **all** files, including hidden ones, use:

```text
ls -la
```

This will also show two special entries:

* `.` → the current directory
* `..` → the parent directory

These entries are important for navigation.

## Listing Other Directories

You do not need to change into a directory to view its contents. You can pass a path directly to `ls`.

```text
ls -l /var/
```

This is useful when inspecting system directories without leaving your current location.

## Changing Directories

To move between directories, you use the `cd` command.

To jump directly to a directory using an absolute path:

```text
cd /dev/shm
```

Your prompt will update to reflect the new location.

### Returning to the Previous Directory

You can quickly return to the last directory you were in using:

```text
cd -
```

This is extremely useful when switching back and forth between two locations.

## Using Tab Completion

The shell provides auto-completion to speed up navigation and reduce typing errors.

If you type part of a path and press `TAB`, the shell attempts to complete it. Pressing `TAB` twice shows all possible matches.

Example:

```text
cd /dev/s<TAB><TAB>
```

Output:

```text
shm/  snd/
```

If the remaining characters uniquely identify a directory, the shell will complete it automatically.

## Parent and Current Directories

As mentioned earlier:

* `.` refers to the current directory
* `..` refers to the parent directory

You can move to the parent directory using:

```text
cd ..
```

This works regardless of where you are in the filesystem.

## Clearing the Terminal

As you work, your terminal can become cluttered. You can clear the screen using:

```text
clear
```

Or by using the keyboard shortcut:

```text
Ctrl + L
```

Both achieve the same result.

## Command History

The shell keeps a history of previously executed commands.

* Use the **up** and **down** arrow keys to scroll through past commands
* Use `Ctrl + R` to search the command history interactively

History search is especially useful when repeating long commands or correcting mistakes.

---

Navigation is something you will do constantly. The more comfortable you become with these commands and shortcuts, the less mental effort basic movement will require. This frees up your attention for more complex tasks later in the path.
