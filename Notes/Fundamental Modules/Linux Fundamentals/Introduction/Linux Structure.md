# Linux Structure

Linux is an operating system you will encounter constantly in cyber security. It runs personal machines, servers, embedded devices, and even mobile platforms. For you as a security practitioner, Linux is not optional knowledge—it is foundational. In this section, you build a mental model of how Linux is put together, why it looks the way it does, and how its design choices affect how you work with it.

You can think of this as your first driving lesson in a new car. Before you drive fast, you need to know where the controls are, what each part does, and why it exists.

## What Linux Is

Linux is an operating system, just like Windows, macOS, iOS, or Android. An operating system is responsible for managing hardware resources and acting as the bridge between software and hardware. Every program you run ultimately depends on the OS to access memory, storage, the CPU, and peripheral devices.

Unlike many commercial operating systems, Linux exists in many different versions called **distributions** (or distros). Each distribution packages the Linux kernel together with tools, libraries, and configuration choices aimed at a specific audience or purpose.

As you work through these modules, you will mainly interact with **Ubuntu**, a Debian-based Linux distribution designed for security, privacy, and development. This environment is provided to you through interactive instances and will be your primary workspace.

## A Short History

Linux did not appear in isolation. Its roots trace back to Unix, developed in the early 1970s. Later efforts such as BSD and the GNU project attempted to build free Unix-like systems, but for a long time there was no widely adopted free kernel.

That changed in 1991 when Linus Torvalds began working on a personal project: a free operating system kernel. Over time, that kernel grew from a small C codebase into a massive, collaboratively developed project with tens of millions of lines of code, released under the GNU General Public License.

Today, Linux exists in hundreds of distributions, including Ubuntu, Debian, Fedora, OpenSUSE, Arch-based systems, Red Hat, and many others. Because Linux is free and open source, anyone can study, modify, and redistribute it. This openness is one of the main reasons Linux dominates servers, cloud infrastructure, and security tooling.

Linux is generally stable, performant, and frequently updated. While it has had vulnerabilities like any other system, its openness and development model allow issues to be discovered and patched quickly. It can feel less beginner-friendly than some alternatives, but that learning curve is exactly what gives you greater control.

## Linux Philosophy

Linux follows a clear and practical philosophy that directly shapes how you interact with the system.

| Principle                     | What it means in practice                                         |
| ----------------------------- | ----------------------------------------------------------------- |
| Everything is a file          | Configuration, devices, and system data are exposed through files |
| Small, single-purpose tools   | Programs do one thing well                                        |
| Tools can be chained together | You combine programs to perform complex tasks                     |
| Avoid captive user interfaces | The terminal gives you direct control                             |
| Configuration stored as text  | System behaviour is transparent and inspectable                   |

You will feel this philosophy most strongly when working in the terminal. Instead of relying on large, opaque tools, you build solutions by connecting simple commands together.

## Core Components

Linux is made up of several key components, each with a specific role:

| Component                            | Purpose                                             |
| ------------------------------------ | --------------------------------------------------- |
| Bootloader                           | Starts the operating system (commonly GRUB)         |
| Kernel                               | Manages CPU, memory, devices, and process isolation |
| Daemons                              | Background services that handle system functions    |
| Shell                                | Command-line interface between you and the OS       |
| Graphics server                      | Enables graphical applications                      |
| Window manager / Desktop environment | Provides the graphical user interface               |
| Utilities                            | User-facing tools and system programs               |

You will spend most of your time interacting with the shell, because that is where Linux gives you the most power and flexibility.

## Linux Architecture

Linux is commonly described in layers. Each layer builds on the one below it.

| Layer            | Role                                          |
| ---------------- | --------------------------------------------- |
| Hardware         | Physical components such as CPU, RAM, storage |
| Kernel           | Virtualises and controls hardware resources   |
| Shell            | Interface for issuing commands                |
| System utilities | Expose OS functionality to the user           |

Understanding these layers helps you reason about where problems occur and which tools are appropriate to use.

## File System Hierarchy

Linux uses a tree-like directory structure defined by the Filesystem Hierarchy Standard (FHS). Everything starts at the root directory `/`.

<img width="2576" height="1656" alt="NEW_filesystem" src="https://github.com/user-attachments/assets/b0e6bcec-21f1-42cf-86e2-9b3d511780f4" />

Below are the most important top-level directories you should recognise immediately:

| Path   | Purpose                                   |
| ------ | ----------------------------------------- |
| /      | Root of the entire filesystem             |
| /bin   | Essential command binaries                |
| /boot  | Bootloader and kernel files               |
| /dev   | Device files representing hardware        |
| /etc   | System-wide configuration files           |
| /home  | Home directories for users                |
| /lib   | Shared libraries required for boot        |
| /media | Mount points for removable media          |
| /mnt   | Temporary mount points                    |
| /opt   | Optional and third-party software         |
| /root  | Home directory for the root user          |
| /sbin  | System administration binaries            |
| /tmp   | Temporary files (often cleared on reboot) |
| /usr   | User binaries, libraries, documentation   |
| /var   | Variable data such as logs and caches     |

You do not need to memorise everything immediately, but you should understand **why** files are placed where they are. This knowledge becomes critical during enumeration, privilege escalation, and incident response.

## Why This Matters for You

Linux is not just another operating system you happen to use. It is the environment where most security tooling lives, where most servers run, and where you will spend a significant amount of your time during assessments.

By understanding Linux structure, philosophy, and layout early, you reduce friction later. Instead of guessing where things might be, you will know where to look, how components interact, and how to reason about unfamiliar systems.

Treat this section as your foundation. Everything that follows builds on it.

