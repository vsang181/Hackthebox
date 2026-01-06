# File System Management

Managing file systems is a core Linux skill. You will deal with it whether you are administering systems, troubleshooting storage issues, or analysing hosts during security assessments. At a high level, file system management is about **how data is stored, organised, accessed, and maintained on disk** 

Linux supports multiple file systems, each designed with different trade-offs. There is no single “best” option. The right choice depends on performance requirements, reliability, compatibility, and workload characteristics.

---

## Common Linux File Systems

You will regularly encounter the following file systems:

* **ext2**
  An older file system without journaling. It has low overhead and is sometimes used on USB drives or embedded systems, but it is not ideal for modern production environments.

* **ext3 / ext4**
  Both support journaling, which helps recover from crashes.
  `ext4` is the default on many Linux distributions because it balances performance, stability, and large file support.

* **Btrfs**
  Designed with advanced features such as snapshots, subvolumes, and built-in integrity checks. It is well suited for complex storage setups and modern workloads.

* **XFS**
  Optimised for high-performance environments and very large files. It is commonly used where high I/O throughput is required.

* **NTFS**
  Originally developed for Windows. It is mainly used for interoperability, such as dual-boot systems or external drives shared between Linux and Windows.

When choosing a file system, you should always consider:

* Performance requirements
* Data integrity and recovery needs
* Compatibility with other systems
* Storage size and scalability

---

## Linux File System Structure

Linux follows the traditional Unix model and uses a **hierarchical directory structure**. Everything starts at the root directory (`/`), and all files and directories branch from there.

A critical concept within this structure is the **inode**.

---

## Inodes

An inode is a data structure that stores **metadata** about a file or directory, including:

* Permissions
* Ownership
* Size
* Timestamps
* Pointers to data blocks

Importantly, **inodes do not store file names or file contents**. File names are managed by directories, which map names to inode numbers.

The **inode table** acts like a database the kernel uses to track all files on the system. This is why you can sometimes run out of inodes even when there is still free disk space.

### Analogy

Think of a library:

* The **books** are the file contents
* The **index cards** are the inodes
* The **catalog** is the inode table

The catalog tells the librarian where each book is stored and who is allowed to access it.

---

## File Types in Linux

You will mainly work with three file types:

### Regular Files

These contain text or binary data, such as scripts, documents, images, or executables. They can exist anywhere in the directory hierarchy, not just in `/`.

### Directories

Directories are special files that store references to other files and directories. Each file has a **parent directory**, which defines where it lives in the hierarchy.

### Symbolic Links

Symbolic links (symlinks) act as references to other files or directories. They allow you to access a target location without duplicating the file itself and are often used to simplify complex directory structures.

---

## Viewing Inodes

You can display inode numbers using the `-i` option with `ls`:

```text
ls -il
```

Example output:

```text
10678872 -rw-r--r-- 1 cry0l1t3 htb 234123 Feb 14 19:30 myscript.py
10678869 -rw-r--r-- 1 cry0l1t3 htb  43230 Feb 14 11:52 notes.txt
```

The first column shows the inode number associated with each file.

---

## Disks and Partitions

Physical storage devices are divided into **partitions**. Each partition can be formatted with a file system and mounted independently.

A commonly used tool for viewing and managing partitions is `fdisk`.

### Viewing Partition Information

```text
sudo fdisk -l
```

This command shows:

* Disk size
* Partition layout
* File system types
* Boot flags

Understanding this output is essential when analysing disk usage or preparing new storage.

---

## Mounting File Systems

To access a partition or device, it must be **mounted** to a directory known as a mount point. Once mounted, the file system becomes part of the directory tree.

### Manual Mount

```text
sudo mount /dev/sdb1 /mnt/usb
```

After mounting, you can interact with the files normally:

```text
cd /mnt/usb
ls -l
```

---

## Automatic Mounting at Boot

Persistent mounts are defined in the `/etc/fstab` file. This file tells the system which file systems should be mounted automatically during boot and with which options.

Example:

```text
/dev/sdb1 /mnt/usb ext4 rw,noauto,user 0 0
```

Key points:

* `noauto` prevents automatic mounting at boot
* `user` allows non-root users to mount the file system

---

## Viewing Mounted File Systems

To list all currently mounted file systems:

```text
mount
```

This output includes:

* Device name
* Mount point
* File system type
* Mount options

This is extremely useful during enumeration and troubleshooting.

---

## Unmounting File Systems

To unmount a file system:

```text
sudo umount /mnt/usb
```

You cannot unmount a file system if it is actively being used.

To check which processes are using files on a mount:

```text
lsof | grep <username>
```

You must stop those processes before unmounting.

---

## Swap Space

Swap space is used as **virtual memory** when physical RAM is exhausted. The kernel moves inactive memory pages to swap, freeing RAM for active processes.

Swap can be implemented as:

* A dedicated partition
* A swap file

### Creating and Enabling Swap

* `mkswap` prepares a device or file for swap
* `swapon` enables it

Swap size depends on workload and available RAM. Modern systems with large memory may use minimal swap, but it is still useful as a safety net.

### Security Considerations

Sensitive data can be written to swap. For this reason, swap space should be **encrypted**, especially on systems handling confidential information.

---

## Swap and Hibernation

Swap is also used for **hibernation**. During hibernation, the system writes its entire memory state to swap and powers off. On resume, the system restores that state from swap.

This means swap must be large enough to hold the contents of RAM if hibernation is enabled.

---

## Why This Matters

File system management directly affects:

* System stability
* Performance
* Data integrity
* Security

As you practise, focus on understanding how storage is structured, how files are tracked, and how access is controlled. These concepts show up repeatedly during real-world assessments and are often the difference between guessing and knowing what is happening under the hood.
