# Anatomy of a Shell

Every operating system exposes its functionality through a shell, the interface through which users and operators issue instructions directly to the operating system kernel. To interact with a shell, a user requires two components working in concert: a terminal emulator application to render the interface, and a command language interpreter to parse and execute the input provided. Understanding both layers is operationally relevant for penetration testers because the tools available post-exploitation are entirely determined by what exists natively on the compromised system.

## Terminal Emulators

A terminal emulator is an application that replicates the function of a physical hardware terminal, providing a graphical or pseudo-terminal interface through which a command language interpreter runs. The emulator itself imposes no constraints on which interpreter it uses; that configuration is independent and user-defined. Common terminal emulators by platform:

| Terminal Emulator | Operating System |
|---|---|
| **[Windows Terminal](https://github.com/microsoft/terminal)** | Windows |
| **[cmder](https://cmder.app/)** | Windows |
| **[PuTTY](https://www.putty.org/)** | Windows |
| **[kitty](https://sw.kovidgoyal.net/kitty/)** | Windows, Linux, macOS |
| **[Alacritty](https://github.com/alacritty/alacritty)** | Windows, Linux, macOS |
| **[xterm](https://invisible-island.net/xterm/)** | Linux |
| **[GNOME Terminal](https://en.wikipedia.org/wiki/GNOME_Terminal)** | Linux |
| **[MATE Terminal](https://github.com/mate-desktop/mate-terminal)** | Linux |
| **[Konsole](https://konsole.kde.org/)** | Linux |
| **[Terminal](https://en.wikipedia.org/wiki/Terminal_(macOS))** | macOS |
| **[iTerm2](https://iterm2.com/)** | macOS |

Selection between these tools is largely a matter of personal preference and workflow. On a target system during an engagement, the available terminal emulator is determined entirely by what the system has installed; operator preference is irrelevant in that context.

## Command Language Interpreters

A command language interpreter parses instructions typed into the terminal and issues the corresponding tasks to the operating system for processing. The command-line interface a pentester interacts with is therefore a combination of three elements: the operating system, the terminal emulator, and the command language interpreter. These interpreters are also referred to as shell scripting languages and map directly to **MITRE ATT&CK T1059: Command and Scripting Interpreter**, which classifies their abuse as an execution technique.

The interpreter in use on a system dictates which commands and scripts will be recognised and executed successfully. Attempting to run Bash-specific syntax inside a PowerShell session, or vice versa, will produce errors or unexpected behaviour. Identifying the active interpreter on a target is therefore a required step before issuing any post-exploitation commands.

## Hands-on with Terminal Emulators and Shells

The **[MATE Terminal](https://github.com/mate-desktop/mate-terminal)** on Parrot OS provides a practical demonstration of how these components interact. Opening it reveals the active interpreter immediately through the shell prompt character; the `$` sign indicates a POSIX-compatible shell such as Bash, Ksh, or a similar variant is running. An unrecognised command produces an interpreter-specific error, confirming which language is active.

### Shell Validation from `ps`

The running interpreter can be confirmed directly by inspecting active processes:

```bash
ps

    PID TTY          TIME CMD
   4232 pts/1    00:00:00 bash
  11435 pts/1    00:00:00 ps
```

The output shows **Bash** running on the current pseudo-terminal (`pts/1`), confirming the active command language interpreter without ambiguity.

### Shell Validation Using `env`

Environment variables provide an alternative confirmation method:

```bash
env

SHELL=/bin/bash
```

The `SHELL` variable explicitly names the interpreter binary in use. This is particularly useful when the prompt character alone is ambiguous, for instance when a custom prompt configuration has removed or replaced the default `$` or `#` symbols.

### PowerShell vs. Bash

Running **[PowerShell](https://learn.microsoft.com/en-us/powershell/)** and Bash side-by-side within the same terminal emulator application illustrates a critical point: the emulator is interpreter-agnostic. The same MATE Terminal instance can run either, and the experience differs entirely based on the interpreter loaded; available commands, output formatting, scripting syntax, and error messages are all interpreter-specific. On a Windows target, `cmd.exe` and PowerShell are the native interpreters; on Linux, Bash is the most prevalent default, though `sh`, `zsh`, and `fish` may also be present depending on the system configuration. Identifying which interpreter is available on a target directly determines which shell commands, one-liners, and scripts can be used effectively during exploitation and post-exploitation.
