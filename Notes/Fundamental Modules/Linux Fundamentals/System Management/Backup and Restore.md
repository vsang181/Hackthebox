# Backup and Restore

Backing up and restoring data is a fundamental responsibility when working with Linux systems. Whether you are administering servers, managing your own workstation, or analysing systems from a security perspective, you must assume that **data loss will eventually happen**. Backups are what turn that assumption into a manageable risk 

Linux provides several mature and reliable tools for creating backups. These tools differ in complexity, flexibility, and security features, allowing you to choose an approach that fits your environment and threat model.

On Ubuntu-based systems, the most commonly used backup solutions include:

* **Rsync**
* **Duplicity**
* **Deja Dup**

Each serves a different purpose, but they are often used together rather than in isolation.

---

## Understanding the Backup Tools

### Rsync

**Rsync** is an open-source utility designed for fast and efficient file synchronisation. Its key advantage is that it only transfers **changed portions of files**, rather than copying everything every time. This makes it extremely efficient for large directories and frequent backups.

Rsync is widely used for:

* Local backups
* Remote backups over the network
* Incremental synchronisation
* Mirroring directories between systems

Because of its efficiency and flexibility, rsync is one of the most important tools to understand.

---

### Duplicity

**Duplicity** builds on top of rsync by adding **encryption and compression**. It is designed for scenarios where backups are stored off-host, such as on remote servers, FTP locations, or cloud storage.

With duplicity, your backups are encrypted before leaving the system. This ensures that even if the storage location is compromised, the data remains unreadable without the encryption keys.

Duplicity is a good choice when confidentiality is a priority.

---

### Deja Dup

**Deja Dup** provides a graphical interface for backups. Internally, it uses rsync and supports encryption, much like duplicity, but hides most of the complexity from the user.

This makes Deja Dup ideal if you want:

* Simple configuration
* Minimal command-line interaction
* Quick restore workflows

While it is easier to use, the underlying principles remain the same.

---

## Why Encryption Matters

Think of your data as something valuable stored in a safe. Backups are copies of that safe, often stored somewhere else. If those copies are not encrypted, anyone who gains access to them can read the contents.

Encrypting backups adds an additional layer of protection. On Ubuntu systems, encryption can be handled directly by tools such as duplicity or supplemented with technologies like:

* **GnuPG**
* **eCryptfs**
* **LUKS**

From a security perspective, encrypted backups are not optional when dealing with sensitive data.

---

## Installing Rsync

Before using rsync, ensure it is installed:

```text
sudo apt install rsync -y
```

Once installed, you can immediately begin creating backups.

---

## Backing Up a Directory with Rsync

### Basic Backup to a Remote Server

```text
rsync -av /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

What this does:

* `-a` (archive) preserves permissions, ownership, timestamps, and symbolic links
* `-v` (verbose) shows progress

The entire directory is copied to the remote host while preserving file attributes.

---

### Incremental and Compressed Backups

You can extend rsync with additional options:

```text
rsync -avz --backup --backup-dir=/path/to/backup/folder --delete /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

This configuration:

* Enables compression (`-z`) for faster transfers
* Stores incremental backups in a separate directory
* Removes files on the destination that no longer exist at the source

This approach is useful for maintaining clean mirrors while still retaining historical versions.

---

## Restoring Data with Rsync

Restoring data is simply the reverse operation:

```text
rsync -av user@remote_host:/path/to/backup/directory /path/to/mydirectory
```

This copies the backup back into the local directory, restoring files and permissions.

---

## Securing Rsync with SSH

By default, rsync does not encrypt data. To protect data in transit, it is commonly combined with **SSH**.

```text
rsync -avz -e ssh /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

Using SSH ensures:

* Encrypted data transfer
* Protection against interception and tampering
* Strong authentication mechanisms

For most environments, rsync over SSH is the recommended approach.

---

## Automating Backups with Cron

Manual backups are unreliable. Automation ensures consistency.

To automate rsync, you typically:

1. Create a backup script
2. Configure SSH key-based authentication
3. Schedule execution with cron

---

## Setting Up SSH Key Authentication

First, generate an SSH key pair:

```text
ssh-keygen -t rsa -b 2048
```

Accept the default path (`~/.ssh/id_rsa`). You may leave the passphrase empty for automation.

Next, copy the public key to the backup server:

```text
ssh-copy-id user@backup_server
```

This allows passwordless authentication for rsync over SSH.

---

## Creating the Backup Script

Create a script called `RSYNC_Backup.sh`:

```bash
#!/bin/bash

rsync -avz -e ssh /path/to/mydirectory user@backup_server:/path/to/backup/directory
```

Make the script executable:

```text
chmod +x RSYNC_Backup.sh
```

Ensure the script is owned by the correct user to prevent tampering.

---

## Scheduling the Backup with Cron

Edit your crontab:

```text
crontab -e
```

Add the following entry to run the backup every hour:

```text
0 * * * * /path/to/RSYNC_Backup.sh
```

Cron will now execute the script automatically at the defined interval.

---

## Practising Locally

For testing purposes, you do not need a remote server. You can simulate the setup locally.

On a lab system such as Pwnbox:

1. Create two directories:

   * `to_backup`
   * `synced_backup`
2. Use rsync to synchronise them
3. Schedule the task using cron
4. Use `127.0.0.1` as the remote host if needed

This allows you to practise safely without affecting real systems.

---

## Why This Matters

Backups are not just about recovery. From a security perspective, they help you:

* Understand data flow
* Identify sensitive directories
* Detect misconfigured permissions
* Spot insecure storage practices

A well-designed backup strategy improves resilience, reduces risk, and provides confidence that data can be recovered when something goes wrong. As you continue, treat backup and restore processes as **first-class system components**, not afterthoughts.
