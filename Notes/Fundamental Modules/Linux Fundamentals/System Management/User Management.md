# User Management

User management is a core part of working with Linux systems. As you spend more time in the terminal, you will regularly interact with **users, groups, and privileges**, whether you are administering a system or assessing it from a security perspective.

From an administrative point of view, user management is about controlling **who can access what**. From a penetration testing perspective, it is about understanding **how access is enforced**, where it might be misconfigured, and how privilege boundaries can sometimes be crossed.

You will often need to:

* Create or remove user accounts
* Assign users to specific groups
* Execute commands as another user
* Understand which files are protected and why

All of these actions rely on the same underlying mechanisms.

---

## Users, Groups, and Privileges

Linux uses users and groups to enforce access control. Files and directories are owned by a user and a group, and permissions determine what actions are allowed.

Some files are intentionally restricted because they contain sensitive information. A good example is the password shadow file.

### Example: Access Denied as a Regular User

```text
willywonka69@htb[/htb]$ cat /etc/shadow
```

Output:

```text
cat: /etc/shadow: Permission denied
```

The `/etc/shadow` file stores **hashed password data** for all users on the system. For security reasons, it is readable only by the `root` user. Allowing regular users to read this file would expose critical authentication data.

---

## Executing Commands as Another User

Some tasks require higher privileges than those assigned to a normal user. Instead of logging in directly as `root`, Linux provides controlled ways to temporarily elevate privileges.

### Using `sudo`

The `sudo` command allows permitted users to execute a command as another user, usually `root`.

```text
sudo <command>
```

Example:

```text
willywonka69@htb[/htb]$ sudo cat /etc/shadow
```

Output:

```text
root:<SNIP>:18395:0:99999:7:::
daemon:*:17737:0:99999:7:::
bin:*:17737:0:99999:7:::
<SNIP>
```

Here, the same command works because it is executed with elevated privileges.

Using `sudo` instead of logging in as root is considered best practice. It reduces risk, improves auditing, and limits the scope of mistakes. You will explore `sudo` permissions and configuration in more depth later when looking at Linux security.

### Using `su`

The `su` command allows you to switch to another user account entirely by providing that user’s credentials. By default, it switches to `root` if no username is specified.

```text
su
```

Unlike `sudo`, this starts a new shell as the target user.

---

## Common User Management Commands

The following commands form the foundation of user and group management. You should become comfortable with what each one does and when it is appropriate to use it.

| Command    | Description                              |
| ---------- | ---------------------------------------- |
| `sudo`     | Executes a command as another user       |
| `su`       | Switches to another user account         |
| `useradd`  | Creates a new user                       |
| `userdel`  | Deletes a user account and related files |
| `usermod`  | Modifies an existing user account        |
| `addgroup` | Creates a new group                      |
| `delgroup` | Removes a group                          |
| `passwd`   | Changes a user’s password                |

Each of these commands supports multiple options, which you should explore using `man` pages and hands-on practice.

---

## Why This Matters for Security

Understanding user management is critical when assessing a system’s security posture. Many vulnerabilities arise from:

* Excessive group memberships
* Weak `sudo` rules
* Misconfigured file ownership
* Users with more privileges than necessary

By understanding how authentication and permissions are supposed to work, you can more easily spot when something is wrong.
