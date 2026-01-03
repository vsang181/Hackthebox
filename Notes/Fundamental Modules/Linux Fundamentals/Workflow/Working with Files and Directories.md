# Working with Files and Directories

One of the biggest differences between Linux and Windows is how you interact with files. On Windows, you usually rely on graphical tools such as File Explorer. On Linux, the terminal gives you direct, precise control over files and directories using commands. Once you are comfortable with this approach, it is often faster and far more flexible than using a graphical interface 

Working in the shell allows you to access, modify, and manage files without opening editors or clicking through menus. You can selectively edit content, chain commands together, redirect output into files, and automate repetitive tasks. This becomes especially powerful when you are working with many files or performing the same operation repeatedly.

In this section, you will practise the core file and directory operations that you will use constantly.

## Creating Files and Directories

Before running these commands, make sure you are logged into the target system via SSH.

### Creating an Empty File

To create an empty file, you use the `touch` command:

```text
touch <filename>
```

Example:

```text
touch info.txt
```

This creates an empty file named `info.txt` in the current directory.

### Creating a Directory

To create a directory, you use the `mkdir` command:

```text
mkdir <directory_name>
```

Example:

```text
mkdir Storage
```

### Creating Nested Directories

When you need to create multiple directories at once, including parent directories that do not yet exist, you can use the `-p` option:

```text
mkdir -p Storage/local/user/documents
```

This creates the full directory structure in one command.

You can visualise the structure using `tree`:

```text
tree .
```

Example output:

```text
.
├── Desktop
├── Documents
├── Downloads
├── info.txt
├── Music
├── Pictures
├── Public
├── Storage
│   └── local
│       └── user
│           └── documents
├── Templates
└── Videos
```

## Creating Files Using Relative Paths

You do not need to change directories to create a file inside another directory. You can specify the path directly.

The single dot (`.`) represents the current directory.

Example:

```text
touch ./Storage/local/user/userinfo.txt
```

After creating the file, your structure will look like this:

```text
.
├── Desktop
├── Documents
├── Downloads
├── info.txt
├── Music
├── Pictures
├── Public
├── Storage
│   └── local
│       └── user
│           ├── documents
│           └── userinfo.txt
├── Templates
└── Videos
```

## Moving and Renaming Files

The `mv` command is used to move **and** rename files or directories.

### Syntax

```text
mv <source> <destination>
```

### Renaming a File

To rename `info.txt` to `information.txt`:

```text
mv info.txt information.txt
```

### Moving Files into a Directory

First, create another file:

```text
touch readme.txt
```

Now move both files into the `Storage` directory:

```text
mv information.txt readme.txt Storage/
```

This moves the files into the specified directory in a single command.

## Copying Files

To copy files instead of moving them, use the `cp` command.

Example:

```text
cp Storage/readme.txt Storage/local/
```

This creates a copy of `readme.txt` inside the `local` directory while keeping the original file intact.

You can confirm the result again using:

```text
tree .
```

## Why This Matters

Basic file operations may seem simple, but they form the foundation for everything that follows. During penetration tests, you will constantly create files, move loot, copy scripts, and organise output. Being fluent with these commands saves time and reduces mistakes.

Later sections will build on this by introducing:

* Input and output redirection
* Text processing
* Interactive editors such as `vim` and `nano`

As you practise, aim to understand **what each command does and why**. That understanding will make it much easier to adapt when working in unfamiliar environments.
