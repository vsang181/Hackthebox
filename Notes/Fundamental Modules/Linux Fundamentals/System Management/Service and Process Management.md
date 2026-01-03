# Service and Process Management

Services (often called **daemons**) are background programs that run without direct user interaction. They are responsible for keeping a Linux system functional and for providing additional capabilities such as remote access, logging, web serving, and scheduling. Understanding how services and processes behave is essential when you are administering a system or analysing it from a security perspective 

At a high level, services fall into two categories.

---

## Types of Services

### System Services

These services are required for the operating system to function correctly. They start automatically during the boot process and handle core tasks such as hardware initialisation, logging, networking, and device management.

You can think of these as the engine and drivetrain of a car. Without them, nothing else works.

### User-Installed Services

These services are installed by users or administrators to add extra functionality. Examples include web servers, database servers, and remote access services.

They are not required for the system to boot, but they extend what the system can do. Using the car analogy, these are features like air conditioning or navigation systems.

Daemons are commonly identified by the letter **`d`** at the end of their name, such as `sshd` (SSH daemon) or `systemd`.

---

## Common Goals When Working with Services and Processes

When interacting with services or processes, you usually want to do one or more of the following:

* Start or restart a service or process
* Stop a service or process
* Check the current or previous state of a service or process
* Enable or disable a service at boot time
* Locate a running service or process

Modern Linux systems provide tools that make all of this manageable in a consistent way.

---

## systemd and Process IDs

Most modern Linux distributions use **systemd** as their init system. It is the **first process started during boot** and is assigned process ID **1**.

Every running program on a Linux system is a process and is assigned a unique **Process ID (PID)**. Processes may also have a **Parent Process ID (PPID)**, indicating which process started them.

Process-related information is exposed through the `/proc` filesystem, which contains runtime data about every active process.

---

## Managing Services with `systemctl`

The primary tool for interacting with services on systemd-based systems is `systemctl`.

### Starting a Service

After installing a service such as OpenSSH, you can start it manually:

```text
systemctl start ssh
```

### Checking Service Status

To verify that the service is running and to inspect its state:

```text
systemctl status ssh
```

This output shows whether the service is active, when it was started, the main PID, and recent log messages.

---

## Enabling a Service at Boot

To ensure a service starts automatically after a reboot, enable it:

```text
systemctl enable ssh
```

Once enabled, the service will start during system initialisation.

---

## Verifying Running Processes

You can confirm that a service is running by inspecting active processes:

```text
ps -aux | grep ssh
```

This displays the SSH daemon process and confirms that it is active.

---

## Listing All Services

To list all services managed by systemd:

```text
systemctl list-units --type=service
```

This provides an overview of loaded and active services, which is particularly useful during system enumeration.

---

## Inspecting Service Logs with `journalctl`

If a service fails to start or behaves unexpectedly, logs are often the fastest way to identify the problem.

To view logs for a specific service:

```text
journalctl -u ssh.service --no-pager
```

This displays historical log entries related to the service, including errors, restarts, and connection attempts.

---

## Process States

A process can exist in several states:

* **Running** – actively executing
* **Waiting** – waiting for a resource or event
* **Stopped** – suspended
* **Zombie** – terminated but still listed in the process table

Understanding these states helps you interpret system behaviour and diagnose issues.

---

## Signals and Process Control

Processes are controlled by sending **signals**. You can view all available signals with:

```text
kill -l
```

Some of the most commonly used signals are:

| Signal         | Description                       |
| -------------- | --------------------------------- |
| `SIGHUP` (1)   | Terminal closed                   |
| `SIGINT` (2)   | Interrupt (`Ctrl + C`)            |
| `SIGQUIT` (3)  | Quit                              |
| `SIGKILL` (9)  | Forcefully terminate (no cleanup) |
| `SIGTERM` (15) | Graceful termination              |
| `SIGSTOP` (19) | Stop process (cannot be handled)  |
| `SIGTSTP` (20) | Suspend (`Ctrl + Z`)              |

### Killing a Process

If a process becomes unresponsive, you can terminate it by PID:

```text
kill 9 <PID>
```

This sends `SIGKILL` and immediately stops the process.

---

## Backgrounding and Foregrounding Processes

Sometimes you need to run long tasks without blocking your terminal.

### Suspending a Process

Pressing `Ctrl + Z` suspends the current process:

```text
[Ctrl + Z]
```

### Viewing Suspended Jobs

```text
jobs
```

This lists all background and suspended jobs in the current shell.

### Resuming a Process in the Background

```text
bg
```

The process continues running without user interaction.

### Starting a Process in the Background

You can also start a process directly in the background using `&`:

```text
ping -c 10 www.hackthebox.eu &
```

When the process finishes, the shell notifies you automatically.

---

## Bringing a Process to the Foreground

To interact with a backgrounded process again:

```text
fg <job_id>
```

This returns the process to the foreground.

---

## Executing Multiple Commands

Linux provides several ways to chain commands together.

### Semicolon (`;`)

Executes commands sequentially, regardless of errors:

```text
echo '1'; echo '2'; echo '3'
```

Even if a command fails, the next one still runs.

---

### Double Ampersand (`&&`)

Executes the next command **only if the previous one succeeds**:

```text
echo '1' && ls MISSING_FILE && echo '3'
```

Here, execution stops as soon as an error occurs.

---

### Pipes (`|`)

Pipes pass the output of one command directly into another. They depend on both the success and the output of the previous command.

You have already seen pipes extensively when filtering and processing text.

---

## Why This Matters

Service and process management is not just administrative knowledge. From a security perspective, it allows you to:

* Identify running services and exposed attack surfaces
* Spot unnecessary or misconfigured daemons
* Analyse logs for suspicious behaviour
* Control long-running tasks efficiently

As you practise, focus on understanding **what is running**, **why it is running**, and **who controls it**. This awareness will become increasingly important as you move into deeper system analysis and exploitation scenarios.
