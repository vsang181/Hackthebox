# NFS

Network File System (NFS) is a distributed file system protocol originally developed by Sun Microsystems. Like SMB, its purpose is to provide remote access to files and directories over a network as if they were local. Unlike SMB, NFS is most commonly used in Linux/Unix environments and is built around a different protocol family and operational model.

NFS clients do not “talk SMB”. In practice, this means NFS enumeration and exploitation techniques differ from SMB, and you should treat them as separate service families during reconnaissance.

A key difference between versions is how authentication is handled:

* In **NFSv3**, authentication is largely host-based and relies heavily on client-supplied identity (UID/GID).
* In **NFSv4**, authentication becomes more user-centric and can integrate stronger security controls (for example, Kerberos), similar in concept to how SMB environments rely on user authentication and central identity.

---

## NFS Versions

| Version   | Features                                                                                                                                                                                                    |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **NFSv2** | Older, widely supported, originally operated entirely over UDP.                                                                                                                                             |
| **NFSv3** | Supports larger files and better error handling, but is not fully compatible with NFSv2 clients.                                                                                                            |
| **NFSv4** | Adds stronger security options (including Kerberos), works better through firewalls, reduces reliance on multiple helper services, supports ACLs, introduces stateful operations, and improves performance. |

NFSv4.1 (RFC 8881) extends NFSv4 with features for clustered deployments, including **pNFS** (parallel NFS) for scalable access to distributed storage. A major operational advantage of NFSv4+ is that it typically uses a single well-known port:

* **TCP/UDP 2049** (primary NFS service)

This greatly simplifies firewall traversal compared to older NFS setups that rely more heavily on auxiliary RPC services.

---

## ONC-RPC, rpcbind, and Why Port 111 Matters

NFS is built on **ONC-RPC (also called SUN-RPC)** and uses **XDR** (External Data Representation) for platform-independent data exchange.

Two ports are especially relevant when footprinting NFS:

* **TCP/UDP 111**: `rpcbind` (portmapper)
* **TCP/UDP 2049**: NFS service

In older deployments (especially NFSv3), the server may also expose additional RPC services such as:

* `mountd` (mount service)
* `nlockmgr` (file locking)
* `nfs_acl` (ACL-related RPC program)

This is why NFS scans often show multiple high ports in addition to 111 and 2049.

---

## Authentication Model and the UID/GID Problem

Classic NFS (especially NFSv3) often relies on client-supplied UNIX identity:

* User ID (**UID**)
* Group ID (**GID**)
* Group memberships

The server typically trusts the client to present correct UID/GID values and maps those directly to filesystem permissions. The issue is that UID/GID mappings may not match across systems, and the server does not necessarily validate the identity beyond the RPC layer.

Because of this trust model, NFS should only be used with UID/GID-based authentication inside **trusted networks** or alongside stronger controls (for example, Kerberos in NFSv4, network segmentation, and strict export restrictions).

---

# Default Configuration

NFS exports are configured in:

* `/etc/exports`

This file defines which directories are shared (“exported”), who can mount them (host/subnet restrictions), and what permissions/options apply.

## Exports File

```bash
cat /etc/exports
```

Example contents (including standard comments):

```text
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
```

### Common Export Options

| Option             | Description                                                                          |
| ------------------ | ------------------------------------------------------------------------------------ |
| `rw`               | Read and write permissions.                                                          |
| `ro`               | Read-only permissions.                                                               |
| `sync`             | Synchronous writes (safer, slower).                                                  |
| `async`            | Asynchronous writes (faster, riskier in crash scenarios).                            |
| `secure`           | Require client to use privileged ports (<1024).                                      |
| `insecure`         | Allow client to use non-privileged ports (>1024).                                    |
| `no_subtree_check` | Disable subtree path verification checks.                                            |
| `root_squash`      | Map root (UID/GID 0) to anonymous UID/GID (prevents root-level access from clients). |

---

## Creating an Export (Example)

```bash
echo '/mnt/nfs  10.129.14.0/24(sync,no_subtree_check)' >> /etc/exports
systemctl restart nfs-kernel-server
exportfs
```

Example output:

```text
/mnt/nfs       10.129.14.0/24
```

This exports `/mnt/nfs` to any client in `10.129.14.0/24`. Any host in that subnet can potentially mount the share and inspect its contents, subject to filesystem permissions and export options.

---

# Dangerous Settings

Even though NFS looks “simple”, a few options have a disproportionate security impact:

| Option           | Why It Is Risky                                                                                       |
| ---------------- | ----------------------------------------------------------------------------------------------------- |
| `rw`             | Enables modification of server-side files from the client.                                            |
| `insecure`       | Allows mounts/traffic from non-privileged client ports (>1024), reducing a historical safety barrier. |
| `nohide`         | Exposes nested mounted filesystems under an export, increasing unintended data exposure.              |
| `no_root_squash` | Allows remote root to act as root on the mounted filesystem (high-impact misconfiguration).           |

### Why `insecure` Matters

Historically, requiring privileged ports (<1024) made it harder for unprivileged users to interact with NFS services from a client machine. With `insecure`, clients can use ports above 1024, which can make interaction easier in environments where host controls are weak or where user separation is relied upon incorrectly.

---

# Footprinting the Service

When footprinting NFS, ports **111** and **2049** are the first targets.

## Nmap (rpcbind + NFS)

```bash
sudo nmap 10.129.14.128 -p111,2049 -sV -sC
```

The `rpcinfo` NSE output is especially useful because it lists active RPC programs and the ports they use. This helps confirm what NFS-related services are exposed and whether additional services such as `mountd` and `nlockmgr` are reachable.

## Nmap NFS Scripts

Nmap includes multiple scripts for NFS enumeration, such as listing exports and displaying mountable paths.

```bash
sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049
```

Typical output can include:

* Exported directories (via `nfs-showmount`)
* Directory listings (via `nfs-ls`)
* Filesystem statistics (via `nfs-statfs`)

---

# Enumerating and Mounting NFS Shares

## Show Available Exports

```bash
showmount -e 10.129.14.128
```

Example:

```text
Export list for 10.129.14.128:
/mnt/nfs 10.129.14.0/24
```

## Mount the Share Locally

Create a local mount point:

```bash
mkdir target-NFS
```

Mount the export:

```bash
sudo mount -t nfs 10.129.14.128:/ ./target-NFS/ -o nolock
```

Then inspect contents like a normal directory:

```bash
cd target-NFS
tree .
```

Example structure:

```text
.
└── mnt
    └── nfs
        ├── id_rsa
        ├── id_rsa.pub
        └── nfs.share
```

---

## Inspecting Ownership (Names vs UID/GID)

Once mounted, you can view file ownership in two useful ways.

### List with Usernames and Group Names

```bash
ls -l mnt/nfs/
```

Example:

```text
-rw-r--r-- 1 cry0l1t3 cry0l1t3 1872 Sep 25 00:55 cry0l1t3.priv
-rw-r--r-- 1 cry0l1t3 cry0l1t3  348 Sep 25 00:55 cry0l1t3.pub
-rw-r--r-- 1 root     root     1872 Sep 19 17:27 id_rsa
-rw-r--r-- 1 root     root      348 Sep 19 17:28 id_rsa.pub
-rw-r--r-- 1 root     root        0 Sep 19 17:22 nfs.share
```

### List with Numeric UID/GID

```bash
ls -n mnt/nfs/
```

Example:

```text
-rw-r--r-- 1 1000 1000 1872 Sep 25 00:55 cry0l1t3.priv
-rw-r--r-- 1 1000 1000  348 Sep 25 00:55 cry0l1t3.pub
-rw-r--r-- 1    0 1000 1221 Sep 19 18:21 backup.sh
-rw-r--r-- 1    0    0 1872 Sep 19 17:27 id_rsa
-rw-r--r-- 1    0    0  348 Sep 19 17:28 id_rsa.pub
-rw-r--r-- 1    0    0    0 Sep 19 17:22 nfs.share
```

This matters because NFS frequently trusts UID/GID values. If you know a file is owned by UID 1000 and writable by that user, a mismatched mapping on your local system can affect what you can read or modify. It also explains why the same NFS mount may behave differently across different client systems.

> If `root_squash` is enabled, remote root will be mapped to an anonymous UID/GID and will not automatically have root-level access to the mounted files.

---

# Common Abuse Paths (High Level)

NFS becomes especially interesting when exports are writable or when `no_root_squash` is enabled. Typical risk scenarios include:

* Writable shares that contain scripts executed by cron or automation
* Exported home directories containing SSH keys, shell history, or configuration
* Misconfigured exports that allow privilege escalation through UID/GID trust
* Sensitive keys or backups stored on shares assumed to be “internal only”

---

# Unmounting

After finishing inspection:

```bash
cd ..
sudo umount ./target-NFS
```
