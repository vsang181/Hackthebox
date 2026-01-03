# Cheat Sheet

This cheat sheet is intended to act as a **quick reference** for the commands you will encounter throughout this module. You are not expected to memorise everything here. Instead, use it as a lookup when something feels familiar but you cannot quite recall the exact syntax or purpose.

Over time, many of these commands will become second nature through repeated use.

## Help and Documentation

| Command                       | Description                                     |
| ----------------------------- | ----------------------------------------------- |
| `man <tool>`                  | Opens the manual pages for the specified tool   |
| `<tool> -h` / `<tool> --help` | Displays the help output for a command          |
| `apropos <keyword>`           | Searches manual page descriptions for a keyword |

## User and System Information

| Command    | Description                                                              |
| ---------- | ------------------------------------------------------------------------ |
| `whoami`   | Displays the current username                                            |
| `id`       | Shows user identity, group memberships, and IDs                          |
| `hostname` | Prints or sets the system hostname                                       |
| `uname`    | Displays operating system and kernel information                         |
| `pwd`      | Prints the current working directory                                     |
| `who`      | Displays users currently logged in                                       |
| `env`      | Prints environment variables or runs a command with modified environment |

## Networking and Processes

| Command    | Description                                      |
| ---------- | ------------------------------------------------ |
| `ifconfig` | Displays or configures network interfaces        |
| `ip`       | Manages network interfaces, routing, and tunnels |
| `netstat`  | Displays network connections and status          |
| `ss`       | Inspects active sockets                          |
| `ps`       | Displays running processes                       |
| `lsof`     | Lists open files                                 |
| `kill`     | Sends a signal to a process                      |
| `bg`       | Resumes a job in the background                  |
| `fg`       | Brings a job to the foreground                   |
| `jobs`     | Lists background jobs                            |

## Storage and Hardware

| Command | Description         |
| ------- | ------------------- |
| `lsblk` | Lists block devices |
| `lsusb` | Lists USB devices   |
| `lspci` | Lists PCI devices   |

## User and Group Management

| Command    | Description                                       |
| ---------- | ------------------------------------------------- |
| `sudo`     | Executes a command as another user (usually root) |
| `su`       | Switches to another user account                  |
| `useradd`  | Creates a new user                                |
| `userdel`  | Deletes a user account                            |
| `usermod`  | Modifies a user account                           |
| `addgroup` | Adds a new group                                  |
| `delgroup` | Removes a group                                   |
| `passwd`   | Changes a user password                           |

## Package Management

| Command    | Description                                       |
| ---------- | ------------------------------------------------- |
| `dpkg`     | Installs, removes, and configures Debian packages |
| `apt`      | High-level package management utility             |
| `aptitude` | Alternative interface for package management      |
| `snap`     | Manages snap packages                             |
| `gem`      | Package manager for Ruby                          |
| `pip`      | Package manager for Python                        |
| `git`      | Version control system                            |

## Services and Logs

| Command      | Description                 |
| ------------ | --------------------------- |
| `systemctl`  | Controls systemd services   |
| `journalctl` | Queries the systemd journal |

## File and Directory Management

| Command    | Description                                 |
| ---------- | ------------------------------------------- |
| `ls`       | Lists directory contents                    |
| `cd`       | Changes the current directory               |
| `clear`    | Clears the terminal screen                  |
| `touch`    | Creates an empty file                       |
| `mkdir`    | Creates a directory                         |
| `tree`     | Displays directory contents recursively     |
| `mv`       | Moves or renames files and directories      |
| `cp`       | Copies files or directories                 |
| `nano`     | Terminal-based text editor                  |
| `which`    | Shows the path of a command                 |
| `find`     | Searches for files in a directory hierarchy |
| `updatedb` | Updates the file index database             |
| `locate`   | Searches files using the indexed database   |

## Viewing and Processing Text

| Command  | Description                                       |
| -------- | ------------------------------------------------- |
| `cat`    | Prints file contents                              |
| `more`   | Paged output viewer                               |
| `less`   | Advanced pager with navigation                    |
| `head`   | Prints the first lines of a file or output        |
| `tail`   | Prints the last lines of a file or output         |
| `sort`   | Sorts text output                                 |
| `grep`   | Searches text using patterns                      |
| `cut`    | Extracts sections from lines                      |
| `tr`     | Translates or replaces characters                 |
| `column` | Formats output into columns                       |
| `awk`    | Pattern scanning and text processing              |
| `sed`    | Stream editor for filtering and transforming text |
| `wc`     | Counts lines, words, and bytes                    |

## Permissions and Ownership

| Command | Description                           |
| ------- | ------------------------------------- |
| `chmod` | Changes file or directory permissions |
| `chown` | Changes file owner and group          |

## Data Transfer and Utilities

| Command                  | Description                              |
| ------------------------ | ---------------------------------------- |
| `curl`                   | Transfers data from or to a server       |
| `wget`                   | Downloads files over HTTP(S) or FTP      |
| `python3 -m http.server` | Starts a simple HTTP server on port 8000 |

---

Treat this cheat sheet as a **living reference**. Revisit it often, experiment with commands in a safe environment, and rely on the manual pages to deepen your understanding. Over time, these tools will become part of your everyday workflow.
