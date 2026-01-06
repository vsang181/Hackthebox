# Keyboard Shortcuts

When you spend a lot of time in the Linux terminal, keyboard shortcuts become one of your biggest productivity boosters. Once you build muscle memory for the most important ones, you will type less, move faster, and rely far less on the mouse. Many of these shortcuts are small time-savers on their own, but together they make a noticeable difference.

You should actively practise these until they feel natural.

---

## Auto-Completion

**`[TAB]`**
Triggers auto-completion based on what you have already typed.

You can use this to:

* Complete command names
* Complete file and directory paths
* Show available options when multiple matches exist

If more than one match is possible, pressing `[TAB]` twice usually lists all available options.

---

## Cursor Movement

These shortcuts help you move around the current command line without retyping.

* **`[CTRL] + A`**
  Move the cursor to the beginning of the line.

* **`[CTRL] + E`**
  Move the cursor to the end of the line.

* **`[CTRL] + ← / →`**
  Jump to the beginning of the previous or next word.

* **`[ALT] + B / F`**
  Move backward or forward one word.

These are especially useful when editing long commands.

---

## Erasing Text

Instead of holding backspace, use these shortcuts to delete more efficiently.

* **`[CTRL] + U`**
  Delete everything from the cursor to the start of the line.

* **`[CTRL] + K`**
  Delete everything from the cursor to the end of the line.

* **`[CTRL] + W`**
  Delete the word immediately before the cursor.

---

## Pasting Deleted Text

* **`[CTRL] + Y`**
  Paste the most recently erased text.

This works well together with `[CTRL] + U`, `[CTRL] + K`, and `[CTRL] + W`.

---

## Ending or Interrupting Tasks

* **`[CTRL] + C`**
  Interrupts the current process by sending the `SIGINT` signal.

You commonly use this to stop:

* Long-running scans
* Hung commands
* Accidental command execution

The process is terminated immediately without confirmation.

---

## End-of-File (EOF)

* **`[CTRL] + D`**
  Sends an End-of-File (EOF) signal.

This is often used to:

* Exit interactive shells
* Close STDIN input
* End input streams

In many cases, it behaves like typing `exit`.

---

## Clearing the Terminal

* **`[CTRL] + L`**
  Clears the terminal screen.

This is functionally similar to running the `clear` command, but faster.

---

## Backgrounding a Process

* **`[CTRL] + Z`**
  Suspends the current process by sending the `SIGTSTP` signal.

The process can later be:

* Resumed in the background with `bg`
* Brought back to the foreground with `fg`

This is useful when a command is running and you need your terminal back.

---

## Command History Navigation

* **`[CTRL] + R`**
  Search backwards through command history using a search pattern.

* **`[↑] / [↓]`**
  Cycle through previous and next commands in the command history.

History search is invaluable when repeating or slightly modifying earlier commands.

---

## Switching Between Applications

* **`[ALT] + [TAB]`**
  Switch between open applications in the graphical environment.

This is not terminal-specific, but you will use it constantly when multitasking.

---

## Zooming the Terminal

* **`[CTRL] + [+]`**
  Zoom in.

* **`[CTRL] + [-]`**
  Zoom out.

These shortcuts improve readability when working for long periods or presenting output.

---

## Final Tip

Do not try to memorise everything at once. Pick a few shortcuts, use them deliberately for a few days, then add more. Once they become habits, working in the Linux terminal will feel significantly smoother and more controlled.
