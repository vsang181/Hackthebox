# Permission Management

Permissions in Linux act as **access controls** that determine who can read, modify, or execute files and directories. You can think of them as keys assigned to users and groups, controlling how system resources are accessed and protected. Every action you take on a Linux system is governed by this permission model, which makes understanding it essential for both system administration and penetration testing 

Each file and directory has:

* **An owner (user)**
* **An associated group**
* **A set of permissions** applied separately to:

  * The owner
  * The group
  * All other users

When you create a new file or directory, it is automatically owned by you and assigned to your primary group. This default ownership plays an important role in how access is controlled across the system.

---

## Directory Access and the Execute Permission

Accessing a directory in Linux is not as simple as being able to see its name. To **enter** or **traverse** a directory, you must have the **execute (`x`) permission** on that directory.

You can think of a directory like a hallway:

* **Read (`r`)** → lets you list what is inside
* **Execute (`x`)** → lets you walk through it
* **Write (`w`)** → lets you change what is inside

Without execute permission, you cannot access the directory at all, even if you can see that files exist inside it.

### Example: Permission Denied

```text
ls -al mydirectory/
```

Output:

```text
ls: cannot access 'mydirectory/script.sh': Permission denied
ls: cannot access 'mydirectory/..': Permission denied
ls: cannot access 'mydirectory/subdirectory': Permission denied
ls: cannot access 'mydirectory/.': Permission denied
```

This behaviour highlights an important rule:

> **Execute permission on a directory is required to traverse it, regardless of your user level.**

Execute permission on a directory does **not** mean you can execute files inside it. To run a file, the file itself must also have execute permission.

To create, delete, or rename files inside a directory, you need **write permission on the directory**, not just on the file.

---

## The Linux Permission Model

Linux permissions are based on three basic rights:

* **Read (`r`)**
* **Write (`w`)**
* **Execute (`x`)**

These permissions are assigned separately to:

1. Owner
2. Group
3. Others

You can see this structure clearly using `ls -l`.

### Example

```text
ls -l /etc/passwd
```

Output:

```text
-rwxrw-r-- 1 root root 1641 May  4 23:42 /etc/passwd
```

Breaking this down:

```text
- rwx rw- r--
|  |   |   |
|  |   |   └─ Permissions for others
|  |   └──── Permissions for group
|  └──────── Permissions for owner
└─────────── File type
```

### File Type Indicators

* `-` → regular file
* `d` → directory
* `l` → symbolic link

---

## Changing Permissions with `chmod`

You can modify permissions using the `chmod` command. There are two common approaches:

### Symbolic Mode

Uses letters to represent users and permissions:

* `u` → user (owner)
* `g` → group
* `o` → others
* `a` → all

Example: add read permission for everyone

```text
chmod a+r shell
```

### Octal Mode

Uses numeric values based on binary representation:

| Permission    | Value |
| ------------- | ----- |
| Read (`r`)    | 4     |
| Write (`w`)   | 2     |
| Execute (`x`) | 1     |

Example:

```text
chmod 754 shell
```

This translates to:

| Role   | Binary | Octal | Permissions |
| ------ | ------ | ----- | ----------- |
| Owner  | 111    | 7     | rwx         |
| Group  | 101    | 5     | r-x         |
| Others | 100    | 4     | r--         |

---

## Changing Ownership with `chown`

Ownership controls who the system treats as the file’s primary authority.

### Syntax

```text
chown <user>:<group> <file_or_directory>
```

### Example

```text
chown root:root shell
```

After this change, the file is owned by the `root` user and group.

Ownership is especially important during privilege escalation, where writable files owned by privileged users can lead to system compromise.

---

## Special Permissions: SUID and SGID

Linux supports **special permission bits** that extend beyond standard read, write, and execute permissions.

### SUID (Set User ID)

When set on a file, the program runs with the **permissions of the file owner**, not the user who executed it.

### SGID (Set Group ID)

When set, the program runs with the **permissions of the file’s group**.

These bits are displayed as an `s` in the execute position:

```text
-rwsr-xr-x
```

While SUID and SGID are sometimes necessary, they introduce serious security risks if misused. If a program with SUID privileges can spawn a shell or execute arbitrary commands, it can lead to full system compromise.

Many common exploitation techniques involving SUID binaries are documented on GTFOBins, which you should treat as a reference during assessments.

---

## Sticky Bit

The **sticky bit** is typically applied to directories and controls who can delete or rename files inside them.

When the sticky bit is set:

* Only the **file owner**
* The **directory owner**
* Or **root**

can delete or rename files, even if the directory is writable by everyone.

### Example

```text
ls -l
```

```text
drw-rw-r-t 3 cry0l1t3 cry0l1t3 4096 Jan 12 12:30 scripts
drw-rw-r-T 3 cry0l1t3 cry0l1t3 4096 Jan 12 12:32 reports
```

### Lowercase vs Uppercase Sticky Bit

* **`t` (lowercase)** → execute permission is set
* **`T` (uppercase)** → execute permission is missing

If execute permission is missing, users cannot access the directory contents, even though the sticky bit exists.

A common real-world example of a sticky directory is `/tmp`.

---

## Why This Matters

Permissions are one of the **most common sources of security flaws** on Linux systems. Misconfigured permissions can expose sensitive files, allow unauthorised modifications, or enable privilege escalation.

As you continue, always pay close attention to:

* Who owns a file
* Which group it belongs to
* What permissions are set
* Whether special bits are enabled

Mastering permission management will significantly improve your ability to understand, assess, and exploit Linux systems safely and effectively.
