# Editing Files

Now that you know how to create files and directories, the next step is learning how to **edit** them directly from the terminal. Linux offers multiple text editors, each with different strengths. In this section, you focus on **Nano** first, as it is straightforward and beginner-friendly, before briefly introducing **Vim**, which you will encounter frequently in real environments 

Editing files efficiently is a critical skill. During assessments, you will often need to modify configuration files, adjust scripts, or quickly inspect sensitive system files without relying on a graphical interface.

---

## Editing Files with Nano

Nano is a simple, terminal-based text editor. It does not use modes, which makes it easier to understand when you are starting out.

### Opening or Creating a File

To create and edit a file using Nano, pass the filename as an argument:

```text
nano notes.txt
```

If the file does not exist, Nano creates it automatically. If it already exists, Nano opens it for editing.

Once inside the editor, you can immediately start typing.

At the bottom of the screen, you will see a list of shortcuts. The caret symbol (`^`) represents the **CTRL** key.

---

## Common Nano Shortcuts

Some essential Nano shortcuts you should remember:

* `CTRL + O` → Save (write out) the file
* `CTRL + X` → Exit Nano
* `CTRL + W` → Search for text
* `CTRL + K` → Cut the current line
* `CTRL + U` → Paste (uncut) text

### Searching Inside a File

To search for text:

1. Press `CTRL + W`
2. Enter the search term
3. Press `ENTER`

To jump to the next match, press `CTRL + W` again and hit `ENTER` without typing a new term.

---

## Saving and Exiting

When you are done editing:

1. Press `CTRL + O`
2. Confirm the filename with `ENTER`
3. Press `CTRL + X` to exit Nano

Back in the shell, you can verify the file contents using:

```text
cat notes.txt
```

---

## Why File Contents Matter in Security

Certain system files are particularly valuable during security assessments. One of the most important is:

```text
/etc/passwd
```

This file contains user account information such as:

* Usernames
* User IDs (UIDs)
* Group IDs (GIDs)
* Home directories

Historically, password hashes were stored here, but modern systems keep them in `/etc/shadow`, which is more tightly protected. If permissions on `/etc/passwd`, `/etc/shadow`, or related files are misconfigured, this can lead to serious security issues or privilege escalation opportunities.

As a penetration tester, you must always pay attention to **file permissions and ownership**, as weak access controls often expose sensitive information.

---

## Introduction to Vim

Vim is another terminal-based editor you will encounter frequently. It is an improved version of the older **Vi** editor and is available on almost every Linux system.

Unlike Nano, Vim is a **modal editor**, meaning your keyboard input behaves differently depending on the current mode. This design makes Vim extremely powerful once you are comfortable with it, but it can feel confusing at first.

You can start Vim by simply typing:

```text
vim
```

---

## Vim Modes Overview

Vim operates using several modes. The most important ones are:

| Mode    | Description                                        |
| ------- | -------------------------------------------------- |
| Normal  | Default mode used for navigation and commands      |
| Insert  | Text entry mode                                    |
| Visual  | Used to select and manipulate blocks of text       |
| Command | Executes commands such as save, quit, or search    |
| Replace | Overwrites existing characters                     |
| Ex      | Allows execution of multiple commands sequentially |

When Vim starts, you are usually in **Normal mode**.

To exit Vim:

1. Press `:` to enter Command mode
2. Type `q`
3. Press `ENTER`

---

## Learning Vim Properly

Vim may feel difficult at first, but it rewards practice. Once you are comfortable with it, editing becomes extremely fast and precise.

To practise interactively, Vim provides a built-in tutorial:

```text
vimtutor
```

The tutor walks you through core concepts step by step and is highly recommended. It takes around 25–30 minutes and teaches by doing rather than reading.

---

## Key Takeaway

Nano is excellent for quick edits and early learning. Vim is a long-term investment that pays off massively in speed and efficiency.

You do not need to master Vim immediately, but you **should** become comfortable opening files, navigating text, and exiting without panic. Over time, your editor will become one of your most frequently used tools in the shell.

Practise editing regularly. Like everything else in Linux, confidence comes from repetition.
