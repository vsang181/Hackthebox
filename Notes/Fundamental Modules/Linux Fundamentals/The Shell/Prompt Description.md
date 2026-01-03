# Prompt Description

The Bash prompt is deliberately simple. Its job is to tell you **who you are**, **where you are**, and **whether the system is ready for input**. Every time you see the prompt, the shell is waiting for you to type a command.

By default, the prompt usually displays:

- Your username
- The system’s hostname
- Your current working directory

The cursor appears immediately after the prompt, indicating that input is expected.

A typical prompt layout looks like this:

```
<username>@<hostname><current working directory>$
```

## Home Directory Indicator

When you are in your home directory, it is commonly represented by a tilde (~). This is also the directory you are placed in immediately after logging in.

```
<username>@<hostname>[~]$
```

The symbol at the end of the prompt is important:

- `$` indicates a **regular (unprivileged) user**
- `#` indicates the **root user**

When you switch to the root account, the prompt usually changes automatically:

```
root@htb[/htb]#
```

## Minimal Shell Prompts

During penetration tests, you will often encounter shells that provide very little context. This usually happens when the environment variable responsible for the prompt (`PS1`) is missing or misconfigured.

In these cases, you may only see:

### Unprivileged User Shell

```
$
```

### Privileged (Root) Shell

```
#
```

Even though these prompts are minimal, they still tell you something important: your current privilege level.

## The `PS1` Variable

The appearance of the Bash prompt is controlled by the `PS1` environment variable. You can think of `PS1` as a template that defines what information is shown every time the shell is ready for input.

By modifying `PS1`, you can display additional context such as:

- Username and hostname
- Full or shortened working directory paths
- Date and time
- Exit status of the last command
- Custom symbols or colours

This is especially useful during penetration tests, where situational awareness matters. A well-configured prompt can help you avoid mistakes, such as running commands as root when you did not intend to.

For example, you may choose to display the **full path** instead of just the directory name, or include the **target IP address** to avoid confusion when working across multiple systems.

## Command History and Documentation

Your command history is another valuable resource. Bash stores previously executed commands in the `.bash_history` file located in your home directory.

You can also use tools such as `script` to record terminal sessions, which is extremely helpful for:

- Documentation
- Reporting
- Reproducing attack paths
- Reviewing mistakes

Good prompt configuration combined with proper command logging makes post-engagement reporting far easier.

## Prompt Escape Sequences

Bash provides special escape sequences that you can use inside `PS1` to insert dynamic information into your prompt.

Below are some commonly used ones:

| Escape Sequence | Description                                |
| --------------- | ------------------------------------------ |
| `\d`            | Date (e.g. Mon Feb 6)                      |
| `\D{%Y-%m-%d}`  | Date in YYYY-MM-DD format                  |
| `\H`            | Full hostname                              |
| `\j`            | Number of jobs managed by the shell        |
| `\n`            | New line                                   |
| `\r`            | Carriage return                            |
| `\s`            | Name of the shell                          |
| `\t`            | Time (24-hour format, HH:MM:SS)            |
| `\T`            | Time (12-hour format, HH:MM:SS)            |
| `\@`            | Current time                               |
| `\u`            | Current username                           |
| `\w`            | Full path of the current working directory |

These values are typically configured in shell startup files such as .bashrc.

## Customisation and Scope

You can customise not only the prompt, but also your entire terminal environment. Colour schemes, fonts, and layout choices all influence usability and fatigue during long sessions.

Conceptually, this is no different from using a graphical desktop: you are still logged in as a specific user, on a specific system, inside a specific directory. The shell simply exposes this information more directly and flexibly.

Prompt customisation itself is outside the scope of this module. However, tools such as **bash prompt generators** and **powerline-style prompts** are worth exploring later if you want a more informative or visually structured setup.

For now, the important thing is this:
**always pay attention to your prompt**. It quietly tells you more about your current state than you might realise.

